# Nimbess - Specs
Nimbess is a distributed controller architecture designed to unify network
policy, as well as express network intent across Kubernetes. The docs repo
contains documentation about Nimbess' core design, function, and developer
guides. This is also the place where specs should be written for new features
to be implemented in various Nimbess projects.

## Documentation Overview
The core folder contains all of the fundamental design and developer
guides to get started with Nimbess. To get started, read these:

* [Architecture](core/architecture.md)
* [Functional Design](core/functional-spec.md)
* [Developer Guide](core/developer.md)

## Feature Specs
All new features should include a Feature Spec that details the changes and
impacts of the new feature. Create a folder for each new feature. Follow
the [example](features/example/example.md) template when adding a new feature.

## Contributing
Contributions are welcome using the standard github pull request model. For new
features in Nimbess, please create an issue in this project and assign it to
yourself. Complete the Feature Spec template for your feature and issue a pull
request to this repo.

## License
Nimbess is available under the Apache 2.0 license. See [LICENSE](LICENSE) file
for details.
