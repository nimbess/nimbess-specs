# Nimbess User Guide

## Getting Started

Nimbess consists of the following runtime components:

* Nimbess etcd
* Stargazer
* Nimbess Agent
* BESS

All of the runtime components are containerized with the exception of requiring
the BESS kernel module to be compiled and loaded. Configuration files for
installing each component are available for Kubernetes and will be referenced
in this guide.

## Requirements
Nimbess leverages BESS' ability to use DPDK to create fast path networks.
It is not required to compile DPDK on the host, however it is required to
enable 2MB hugepages on the host. The BESS container will be deployed with
hugepages. At a minimum you should provide at least 1024MB x N sockets on
your host in available hugepages. It is also required to set selinux to
permissive mode.

## Compiling the BESS kernel module

It is required to compile and load the BESS kernel module into the hosts where
Nimbess will run. Execute the following as root on each server:

1. git clone the BESS repo: git clone https://github.com/kot-begemot-uk/bess
2. cd bess
3. Follow the highlighted steps:
<https://github.com/kot-begemot-uk/bess/blob/k8-development/build.sh#L1:L10>
4. ./build.py kmod
5. insmod ./core/kmod/bess.ko

## Deploying Nimbess in Kubernetes

Nimbess requires its own service account to be created with proper RBAC
permissions in order to monitor and update k8s resources. Create the proper
account by running:

```bash
kubectl apply -f https://raw.githubusercontent.com/nimbess/stargazer/master/deployments/nimbess-rbac-ns-account.yaml
```
Next, we need to create the Nimbess etcd, which will will be used to store
Nimbess related information from Stargazer and the Agent:

```bash
kubectl create -f https://raw.githubusercontent.com/nimbess/stargazer/master/deployments/stargazer-etcd.yaml
```
Now install Stargazer:
```bash
kubectl create -f https://raw.githubusercontent.com/nimbess/stargazer/master/deployments/stargazer-pod.yaml
```

Finally, we need to install the Nimbess CNI binary, Agent, and start BESS. The
following yaml file is used to deploy these, however you may need to edit
it first in order to have the proper hugepage settings as referred to in
[Requirements](#requirements). To deploy:

```bash
kubectl apply -f https://raw.githubusercontent.com/nimbess/nimbess-agent/master/k8s/nimbess.yaml
```

If everything deployed correctly your cluster should look something like this:
```bash
[root@trozet ~]# kubectl -n kube-system get pod
NAME                                       READY   STATUS    RESTARTS   AGE
etcd-trozet.localhost                      1/1     Running   0          136m
kube-apiserver-trozet.localhost            1/1     Running   0          136m
kube-controller-manager-trozet.localhost   1/1     Running   0          136m
kube-proxy-87c4z                           1/1     Running   0          137m
kube-scheduler-trozet.localhost            1/1     Running   0          136m
nimbess-agent-7lj9v                        3/3     Running   0          76m
nimbess-etcd                               1/1     Running   0          76m
stargazer                                  1/1     Running   0          63m
```

## Using Nimbess

The following sections will go through some example usages of Nimbess. The
current network design in Nimbess uses L2 networking for pod to pod
connectivity via the BESS data plane forwarder. External/Internet destined
traffic is forwarded to the Linux network stack for processing. In the future,
L3 forwarding will be handled by BESS itself.

### Nimbess Unified Network Policy

To exercise Nimbess UNP, the following example will walk through adding a URL
Filter to the default k8s network. First create a pod that will serve as an
HTTP client:

```bash
kubectl create -f https://github.com/nimbess/nimbess-agent/blob/master/k8s/pod_example.yaml
```

Next we will create a webserver, but first create the content for the website:
1. Create a directory to hold our website:
```bash 
mkdir -p /tmp/website/blockthis/
```
2. Create the HTTP content:
```bash
cat > /tmp/website/blockthis/bad.html << EOI
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Fake News</title>
</head>
<body>
    <h1>shouldnt work</h1>
</body>
</html>
EOI
```
3. Now create the webserver pod:
```
cat << EOF | kubectl apply -f -
---
apiVersion: v1
kind: Pod
metadata:
  name: webserver1
spec:
  containers:
  - name: webserver1
    image: docker.io/httpd
    volumeMounts:
    - mountPath: /etc/podinfo
      name: podinfo
      readOnly: false
    - mountPath: /var/lib/nimbess/cni/
      name: shared-dir
    - mountPath: /usr/local/apache2/htdocs/
      name: web-dir
  volumes:
  - name: podinfo
    downwardAPI:
      items:
        - path: "labels"
          fieldRef:
            fieldPath: metadata.labels
        - path: "annotations"
          fieldRef:
            fieldPath: metadata.annotations
  - name: shared-dir
    hostPath:
      path: /var/lib/nimbess/cni/
  - name: web-dir
    hostPath:
      path: /tmp/website/
EOF
```

4. Now find the webserver IP address, which in this example is 40.0.0.5.

```bash
[root@trozet ~]# kubectl get pod -o wide
NAME         READY   STATUS    RESTARTS   AGE   IP         NODE               NOMINATED NODE   READINESS GATES
pod1         1/1     Running   0          42m   40.0.0.4   trozet.localhost   <none>           <none>
webserver1   1/1     Running   0          13m   40.0.0.5   trozet.localhost   <none>           <none>
```

5. Test that we can now curl the forbidden URL from pod1:

```bash
[root@trozet ~]# kubectl exec -it pod1 curl http://40.0.0.5/blockthis/bad.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Fake News</title>
</head>
<body>
    <h1>shouldnt work</h1>
</body>
</html>

```

6. Now let's define a Nimbess UNP in order to block users from being able to
reach this URL (replace the IP with your webserver's IP):

```bash
cat << EOF | kubectl apply -f -
---
apiVersion: nimbess.com/v1
kind: UnifiedNetworkPolicy
metadata:
  name: testpolicy
  namespace: kube-system
spec:
  l7Policies:
    - default:
        action: allow
      urlFilter:
        action: deny
        urls:
          - http://40.0.0.5/blockthis/bad.html
        podSelector:
          matchLabels:
            environment: production
        network: k8s_pod_network
  podSelector:
    matchLabels:
      environment: dev
  network: devNetwork
EOF
```

7. Verify the URL is now blocked:

```bash
[root@trozet ~]# kubectl exec -it pod1 curl  http://40.0.0.5/blockthis/bad.html
curl: (56) Recv failure: Connection reset by peer
command terminated with exit code 56
```

### Nimbess Fast Path

When Nimbess CNI is invoked, it always creates 2 ports. The first is a kernel
based connection to the default k8s network, while the second port is deemed
as the port to use for high throughput applications aka Fast Path. Currently
the only supported Fast Path is via DPDK. Each pod that is created will receive
a DPDK vhost socket port as well as connectivity to the default k8s network.

The downward API is used in order to pass the DPDK based port information into
the pod. For example, creating the example pod:

```bash
cat << EOF | kubectl apply -f -
---
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: pod1
    image: docker.io/centos/tools:latest
    command:
      - /sbin/init
    volumeMounts:
    - mountPath: /etc/podinfo
      name: podinfo
      readOnly: false
    - mountPath: /var/lib/nimbess/cni/
      name: shared-dir
  volumes:
  - name: podinfo
    downwardAPI:
      items:
        - path: "labels"
          fieldRef:
            fieldPath: metadata.labels
        - path: "annotations"
          fieldRef:
            fieldPath: metadata.annotations
  - name: shared-dir
    hostPath:
      path: /var/lib/nimbess/cni/
EOF
```
This pod will have its Fast Path interface info stored in /etc/podinfo/annotations
along with its vhost socket mounted under /var/lib/nimbess-cni. A DPDK
application can then leverage this information to configure this DPDK interface.

Today the Fast Path interface is connected to a default Fast Path network. In
the future this behavior will be deprecated for explicitly configuring which
network to use via a UNP and Network Attachment Definition.
