- Feature Name: `package-structure`
- Start Date: 2024-10-24
- RFC PR: [CMIP-REF/rfcs#0001](https://github.com/CMIP-REF/rfcs/pull/0001)

# Summary
[summary]: #summary

This RFC proposes how we should structure the git repository to enable the rapid development of the project while it is immature.
The core of the project will have some reuse outside of the REF frontend
so it should be publishable as a separate package.
The goal is to make it easier for users to extend the project with new metrics providers.

# Motivation
[motivation]: #motivation

* At the start of this project we will be in a state of flux until we have a working prototype.
 APIs will change, and we will be adding and removing features.
* Each metrics package will come with its own set of dependencies. Odds are that they will clash with other packages.
* We want something that is easy to extend in future, incorporating new metrics providers/metrics.
* Consistent tooling across the project

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

It is proposed that we start with a mono-repo structure consisting of multiple packages,
each with their own set of dependencies.
The tooling and CI will be shared across all projects.
There will be a `packages` directory at the root of the repo, with each package in its own subdirectory.
An example has been implemented in [CMIP-REF/cmip-ref#1](https://github.com/CMIP-REF/cmip-ref/pull/0001),
which contains a `ref-core` package and an example `ref-metrics-example` package.

Each package will have a `README.md` file that describes the purpose of the package and how to use it,
its own `pyproject.toml` describing the package and it's dependencies, 
and their own set of tests.
The test suite for each package is run with only the dependencies required for that package.

A common set of integration tests will also be run to ensure that the packages work together.

The purpose of this structure is to split the project into smaller, more manageable parts
and to differentiate between the "library" part of the project which is reusable
and the "application" which builds ontop of the library + metrics packages + additional services.

An additional benefit is that it will be easier to refactor code if it is all accessible in the same repository.

## Packages
### `ref-core`
This package will be a library containing the core functionality of the REF.
This package will be a dependency of all other packages as it will describe the interfaces that metrics providers must implement. 
This allows us to keep the core functionality separate from the metrics providers.

### `ref-metrics-*`
Each metric provider will implement a separate package that implements the interfaces described by the `ref-core` package.
These will very thin packages that wrap around the existing APIs of the benchmarking packages or use CMEC to execute.

An example package `ref-metrics-example` has been implemented in [CMIP-REF/cmip-ref#5](https://github.com/CMIP-REF/cmip-ref/pull/5).
See the [code](https://github.com/CMIP-REF/cmip-ref/tree/basic-interface/packages/ref-metrics-example) for more details.

The purpose of these packages are to provide a Python-based hook to execute a metric and return the results.
This will include (hopefully thin) wrappers around the metric providers existing APIs
or build the required CMEC configuration files.
It should require little to no change to the existing codebase benchmarking packages.

The repository will set up a CI pipeline that will run the unit tests for each package.
This includes fetching some sample data.

These packages will maintain their own dependencies (python and otherwise) and packaged up as a docker image.

The package will be named `ref-metrics-<provider>` where `<provider>` is the name of the provider.

### `ref-api` (TBD)
This application package will bring together the core and metrics packages to provide a service that
responds to new data submissions,
tracks the execution of metrics and the results.
The package may have indirect dependencies on the metrics packages to avoid conflicting dependencies.
That adds additional complexity, but will be more flexible in the long run.

This is the service that will be deployed to ESGF alongside any additional services that are required.

Whether this application can be done solely via a CLI-based tool is up for debate.

# Drawbacks
[drawbacks]: #drawbacks

* More people working in the same repository may lead to more merge conflicts.
* Larger repository with more moving parts/concepts


Another drawback is that it will be harder to manage different package managers.
The project currently uses [uv](https://docs.astral.sh/uv/) as a package manager as the core and framework
will not require any complex to install dependencies, 
but science packages may require more complex dependencies.
There is nothing stopping the use of something like [pixi](https://pixi.sh/dev/) when conda-based
dependencies are required for a particular package.

The CI times may become longer. This may be somewhat insidious
as at the beginning the developer experience will be great.
This will gradually degrade as the number of packages/providers increases.
The test for each provider can be run as a separate job.
We may also choose to target a single Python version if it becomes an issue.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

A mono-repo structure will enable the rapid development of the project while it is immature
and in a state of flux.
It will be easier to refactor code as it is all in the same repository.

- Splitting out each package to a separate repository is a possibility
and is a very common approach in the Python community.
The tooling for managing mono-repos is more mature than it was a few years ago which enables the proposed approach.
The downside of that is that changes across multiple repositories will be required as we add new features.
- A single package with extras.
The downside of this is that it will be harder to incorporate new providers as they will be incorporated
into the main package.
The framework will be an application rather than a library so pip installing it will not be as useful.

# Prior art
[prior-art]: #prior-art

I've tried this out in another [recent project](https://github.com/climate-resource/bookshelf) which 
published multiple packages to PyPI from a single repository.
It worked well for that project as it enabled clearer separations and reuse of parts of the codebase
without depending on a much larger package.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* An example that uses `pixi` to install conda packages would be useful as a proof of concept.

# Future possibilities
[future-possibilities]: #future-possibilities

Once the project has matured, we can consider splitting the packages into separate repositories.
Each package can then have its own tooling and versioning.
That would depend on packaging and publishing the `ref-core` package to PyPI and conda-forge.
