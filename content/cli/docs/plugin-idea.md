---
layout: docs
title: IDEA Plugin
description: The IDEA plugin generates classpath information for IntelliJ IDEA projects that reflect the project's dependencies.
weight: 62
plugin: true
---

## Overview

The IDEA plugin allows you to update your IntelliJ IDEA module file (IML) when your project's dependencies change. This plugin does not currently generate IntelliJ IDEA project files (IPR).

**LATEST VERSION: 0.1.5**

## Loading the plugin

Here is how you load this plugin:

~~~~ groovy
idea = loadPlugin(id: "org.lattejava.plugin:idea:0.1.5")
~~~~ 

## Settings

The IDEA plugin provides a couple of settings that help you control how it behaves. None of these settings are required.

### IML file
The first setting changes the `.iml` that the plugin updates. Some projects have different names than the `.iml` file and this setting allows you to configure the file to use. Here is an example:

~~~~ groovy
idea.settings.imlFile = "foobar.iml"
~~~~ 

You can also specify the setting using the Java `Path` object like this:

~~~~ groovy
idea.settings.imlFile = Paths.get("foobar.iml")
~~~~ 

### Module dependencies

The other setting is more complex but very powerful. If you have an IntelliJ IDEA project that consists of multiple modules and there are inter-module dependencies, IntelliJ allows you to specify module-to-module dependencies. This helps with compilation and refactoring. However, Latte does not have the knowledge of inter-project dependencies except via **integration builds** and standard project dependencies. Therefore, you need to tell the IDEA plugin which of your project's dependencies are actually module dependencies in the `.iml` file.

This is done like this:

~~~~ groovy
idea.settings.moduleMap = [
    "com.mycompany:some-other-project:1.0.0-{integration}" : "some-other-project"
]
~~~~ 

What this tells the plugin is that the dependency `com.mycompany:some-other-project:1.0.0-{integration}` is really on the project module `some-other-project`.

### Dependency groups

The final setting you can change is the dependenciesMap setting. This setting controls how Latte dependency groups map to IntelliJ IDEA scopes and how dependencies are fetched. Here is the default value for this setting:

~~~~ groovy
idea.settings.dependenciesMap = [
    "PROVIDED": [
        [group: "provided", transitive: true, fetchSource: true, transitiveGroups: ["provided", "compile", "runtime"]]
    ],
    "COMPILE": [
        [group: "compile", transitive: false, fetchSource: true]
    ],
    "TEST": [
        [group: "test-compile", transitive: true, fetchSource: true, transitiveGroups: ["compile", "runtime"]],
        [group: "test-runtime", transitive: true, fetchSource: true, transitiveGroups: ["compile", "runtime"]]
    ],
    "RUNTIME": [
        [group: "compile", transitive: true, fetchSource: true, transitiveGroups: ["compile", "runtime"]],
        [group: "runtime", transitive: true, fetchSource: true, transitiveGroups: ["compile", "runtime"]]
    ]
]
~~~~ 

This setting tells the plugin that the IntelliJ IDEA scope `PROVIDED` should map to the Latte dependency group `provided` and that the plugin should fetch transitive dependencies in the groups `provided`, `compile`, and `runtime`. Additionally, the IntelliJ IDEA scope `COMPILE` should map to the Latte dependency group `compile` and that transitive dependencies for this group should NOT be fetched. Furthermore, all of these mappings instruct the plugin to always fetch source JARs. This allows IntelliJ IDEA to index the source so that you can jump between your code and the source for the libraries you use.

## Updating the IML file

This plugin provides a single method to update the project's `.iml` file. Here is how you call this method:

~~~~ groovy
idea.iml()
~~~~ 

The `iml` method also accepts an optional closure that receives the root Groovy `Node` of the parsed `.iml` XML, which you can use to make additional modifications before the file is written back. For example:

~~~~ groovy
idea.iml { root ->
  def component = root.component.find { it.@name == 'NewModuleRootManager' }
  component.appendNode("orderEntry", [type: "inheritedJdk"])
}
~~~~ 
