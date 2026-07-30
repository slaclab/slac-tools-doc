# slac-tools Developers Guide

This page is meant to help existing high level applications (HLA) developers of [slac-tools](https://github.com/slaclab/slac-tools). Contributions are welcome. To update this or other documents on slac-tools-doc, please try to follow these steps:
1. Find or write an issue related to your changes.
2. Start a branch from that issue page.
3. Update the branch with your changes.
4. Submit a PR for your branch. 

These steps should be followed when contributing to [slac-tools](https://github.com/slaclab/slac-tools) as well.

## Python versions 

By default, we support stable versions of Python (not feature versions). As versions are declared end of life by [python.org](https://www.python.org/downloads/), tests and workflows should be updated accordingly. 

## Python style
Please try to follow the [PEP 8 Style Guide](https://peps.python.org/pep-0008/). 

## Dependancies
New dependancies should meet the following requirements:
- The repository is hosted by an organization or group
- The repository ... 
- TODO

## Where to develop

Whether you develop on your local laptop or SLAC dev systems, please try to develop code on branches that are associated with issues. You can create a branch from an issue on the lower right hand side of the issue page. 

<img width="1732" height="938" alt="create-issue-branch" src="https://github.com/user-attachments/assets/99257d5b-52ee-421c-8ea9-495c616f2be5" />

## Linting 
TODO: package, how to install/run locally

## Data locations

On production, HLA applications should save data in <$PHYSICS_DATA/<app>/>. 
