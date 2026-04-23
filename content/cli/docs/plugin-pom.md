---
layout: docs
title: POM Plugin
description: The POM plugin generates and updates Maven POM files from your Latte project.
weight: 63
plugin: true
---

## Overview

The POM plugin generates and updates Maven `pom.xml` files from your Latte project's dependencies. This is useful when you need to publish your project to a Maven repository or provide Maven compatibility.

**LATEST VERSION: 0.1.5**

## Loading the plugin

Here is how you load this plugin:

~~~~ groovy
pom = loadPlugin(id: "org.lattejava.plugin:pom:0.1.5")
~~~~

## Settings

The POM plugin has one main setting that controls how Latte dependency groups are mapped to Maven scopes. The settings for the plugin are configured via the `POMSettings` class. The `POMPlugin` class has a field named `settings` that is the main instance of this class.

### Group to scope mapping

You can configure how Latte dependency groups map to Maven scopes and the optional flag using the `groupToScope` field:

~~~~ groovy
pom.settings.groupToScope = [
  "compile": [scope: "compile", optional: false],
  "compile-optional": [scope: "compile", optional: true],
  "provided": [scope: "provided", optional: false],
  "runtime": [scope: "runtime", optional: false],
  "test-compile": [scope: "test", optional: false],
  "test-runtime": [scope: "test", optional: false]
]
~~~~

This is the default mapping. Each key is a Latte dependency group name, and the value is a Map containing the Maven `scope` and `optional` flag.

## Updating the POM

The `update` method generates or updates the project's `pom.xml` file based on the current project dependencies:

~~~~ groovy
pom.update()
~~~~

If `pom.xml` does not exist, it will be created. If it already exists, the dependencies section will be updated.

You can also pass an optional closure that receives the root XML element for custom modifications:

~~~~ groovy
pom.update { root ->
  // Add custom XML elements to the POM
}
~~~~
