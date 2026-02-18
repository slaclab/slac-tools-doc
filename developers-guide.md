# slac-tools Developers Guide

This page is meant to help existing and new high level applications (HLA) developers of [slac-tools](https://github.com/slaclab/slac-tools). Contributions are welcome. To update this or other documents on slac-tools-doc, please follow these steps:
1. Find or write an issue related to your changes
2. Start a branch from that issue page
3. Update the branch with your changes
4. Submit a PR for your branch. 

These steps should be followed when contributing to [slac-tools](https://github.com/slaclab/slac-tools) as well.

## Python versions 

By default, we should support stable versions of Python (not feature versions). As versions are declared end of life by [python.org](https://www.python.org/downloads/), tests and workflows should be updated accordingly. 

## Python style
Please try to follow the [PEP 8 Style Guide](https://peps.python.org/pep-0008/). 

## Where to develop

Please develop on branches that are associated with issues. You can create a branch from an issue on the lower right hand side of the issue page. 

## Development locations

On production, HLA applications should save data in <$PHYSICS_DATA/<app>/>. 