---
layout: docs
title: Dependency Management
description: Latte provides a complete SemVer compliant dependency management system.
weight: 30
---

## Overview

At its core, Latte is a dependency management tool. The build system is built on top of this dependency management. If you aren't familiar with dependency management, the simple explanation is that any libraries or frameworks your project uses are its dependencies. Those libraries and frameworks might also have dependencies of their own. This collection of dependencies forms what is called a dependency graph. Latte is really good at managing dependency graphs and downloading and caching your project's dependencies.

## Definitions

* **Artifact** - a file that a project publishes that can be run or used by other projects (i.e. JAR files, WAR files, SQL files, tarballs, etc)
* **Dependency** - when a project depends on another project's artifact
* **Version** - the version of an artifact
* **Release** - when a project's development has finished and its artifacts have been produced and published
* **Workflow** - the steps used to download and cache dependencies
* **Integration Build** - when a project is in development prior to being released but its artifacts are needed to continue development or testing of another project. Many other build tools use the term SNAPSHOT for this concept.

At different stages of your project's build and run process, it might need different dependencies. For example, at compile time it might need Apache Commons Collections version 3.1. On the other hand, at test time it might require TestNG version 4.8.7. Therefore, dependencies are broken up into groups, one for each stage of the project build and run process.


## Semantic versioning

Latte's dependency management system implements [Semantic Versioning](http://semver.org). Semantic Versioning is a specification that determines how projects must maintain binary compatibility. If you haven't read the Semantic Versioning specification, it is probably a good idea to start there.

Latte does deviate from the Semantic Versioning specification in one major way. Latte uses a version marker for integration builds. The marker is `-{integration}`. This marker is automatically handled by Latte. Although, if you have a dependency on a project that is an integration build, you'll need to add the marker to the dependency declaration.


## Licenses

Latte also requires that all projects define the license(s) they are using. This is a vital piece of data for many organizations to track and maintain and therefore all artifacts must be published with license information. Latte handles this process automatically by requiring that all projects declare their license(s) in the Latte project file.
