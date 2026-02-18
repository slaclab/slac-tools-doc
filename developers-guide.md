# slac-tools Developers Guide

This page is meant to help existing and new high level applications (HLA) developers of slac-tools.

## Python versions

By default, we should support stable versions of Python (not feature versions). As versions are declared end of life by [python.org](https://www.python.org/downloads/), tests and workflows should be updated accordingly. 

## Creating branches

Please develop on branches that are associated with issues. You can create a branch from an issue on the lower right hand side of the issue page. 

## Development locations

On production, HLA applications should save data in <$PHYSICS_DATA/<app>/>. 