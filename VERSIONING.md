# Versioning Policy

The [semantic versioning][semver] specification can be broadly summarized as the following:

> Given a version number MAJOR.MINOR.PATCH, increment the:
> 1) MAJOR version when you make incompatible API changes
> 2) MINOR version when you add functionality in a backward compatible manner
> 3) PATCH version when you make backward compatible bug fixes

ibc-rs adheres to this scheme in spirit, but makes a few distinctions / exceptions:
1) The PATCH version is incremented whenever backward-compatible changes are made, whether they're bug fixes or new features.
2) The MINOR version is incremented when API-breaking and / or consensus-breaking changes are introduced.
3) The MAJOR version is incremented when

[semver]: https://semver.org/
