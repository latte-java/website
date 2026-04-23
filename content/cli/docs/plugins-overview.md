---
layout: docs
title: Plugins Overview
description: Latte plugins provide re-usable build logic but not build targets.
weight: 40
plugin: true
---

## Overview

Latte uses a different approach to plugins than most other build tools. Latte's approach is similar to the approach used by Ant. Latte plugins provide reusable logic that project files can leverage. However, plugins do not provide build targets.

# Why no build targets?

The decision to move build targets out of plugins and back into the project files was intentional. Build systems that allow plugins to define build targets often run into the issue of plugin dependencies.

Here's a simple example. Let's say you have a Java project that you want to compile and test. In Gradle or Maven, you might use the **Java** plugin and the **JUnit** plugin. The **Java** plugin creates these build targets in your project:

* `clean`
* `build`
* `jar`

The **JUnit** plugin creates this build target:

* `test`

The `test` build target will naturally need to ensure that all the source files are compiled first. There are a few ways to achieve this:

1. The **JUnit** plugin must depend on (or extend) the **Java** plugin
2. The build system can define build stages and each plugin can "attach" to different stages

Solution #1 quickly becomes messy and difficult to manage. What if you want to inject some new build logic between the `build` build target and the `test` build target? This is challenging because there is a strong coupling between the **Java** and **JUnit** plugins that is challenging to modify.
 
Solution #2 is equally cumbersome. Each plugin now has to add its build targets to different stages. However, the order of targets within stages might be important but could be challenging to define and manage. Stages are often arbitrarily named and might not map to your project's needs. Sometimes you end up creating new stages just to get things to execute in the correct order.

Rather than try to solve these issues and add massive amounts of complexity, Latte moves the build targets back into the project file and allows the developer to define the order build targets are run in as well as the dependencies between the targets.

# Plugin methods

Latte plugins are simple objects. A project file can invoke methods on the object to perform a set of tasks. This method makes it simple to write project files using Latte plugins.

Here is an example of how simple it is to compile Java code using Latte:

~~~~ groovy
java = loadPlugin("org.lattejava.plugin:java:2.2.0")
java.settings.jdkVersion = "17"

target(name: "build") {
  java.compile()
}
~~~~

That's it! It couldn't be simpler.

# Finding plugin methods and settings

There is currently no built-in command to enumerate the methods or settings of a loaded plugin. Use the plugin pages in this documentation as the reference — each plugin page lists its settings, methods, and any plugin-specific configuration.

The list of first-party plugins is in the sidebar: [Java](../plugin-java/), [Groovy](../plugin-groovy/), [Java TestNG](../plugin-java-testng/), [Groovy TestNG](../plugin-groovy-testng/), [File](../plugin-file/), [Database](../plugin-database/), [Release Git](../plugin-release-git/), [Dependency](../plugin-dependency/), [Linter](../plugin-linter/), [Debian](../plugin-debian/), [IDEA](../plugin-idea/), and [POM](../plugin-pom/).
