---
layout: docs
title: Groovy TestNG Plugin
description: The Groovy TestNG plugin allows you to run TestNG tests for your Groovy project.
weight: 53
plugin: true
---

## Overview

The Groovy TestNG plugin allows you to execute TestNG tests in a Groovy project.

**LATEST VERSION: 0.1.0**

## Loading the plugin

Here is how you load this plugin:

~~~~ groovy
groovyTestNG = loadPlugin(id: "org.lattejava.plugin:groovy-testng:0.1.0")
~~~~ 

## Settings

The Groovy TestNG plugin requires a couple of configuration settings, including the Groovy version and the Java version to execute the tests with. The settings for the plugin are configured via the `GroovyTestNGSettings` class. The `GroovyTestNGPlugin` class has a field named `settings` that is the main instance of this class. Here is an example of configuring the plugin:

~~~~ groovy
groovyTestNG.settings.groovyVersion = "5.0"
groovyTestNG.settings.javaVersion = "17"
~~~~ 

These two properties specify the versions to use, but you must also configure the location of the GDK and JDK on your computer. These locations are configured in two configuration files in your home directory. These files are:

* `~/.config/latte/plugins/org.lattejava.plugin.java.properties`
* `~/.config/latte/plugins/org.lattejava.plugin.groovy.properties`

The first file is used to define the location of the JDKs on your computer. Here's an example of the `~/.config/latte/plugins/org.lattejava.plugin.java.properties` file:

~~~~ 
17=/Users/me/dev/java/current17
21=/Users/me/dev/java/current21
25=/Users/me/dev/java/current25
~~~~ 

Here's an example of the `~/.config/latte/plugins/org.lattejava.plugin.groovy.properties` file:

~~~~ 
4.0=/Users/me/dev/groovy/groovy-4.0.22
5.0=/Users/me/dev/groovy/groovy-5.0.5
~~~~ 

This class has additional fields that allow you to configure the command-line arguments passed to Java when it is run, the verbosity of the TestNG output, and the location of the test reports. These settings are covered below.

### JVM arguments

You can specify additional parameters to the JVM when TestNG executes your tests using the `jvmArguments` field on the settings class. Here is an example:

~~~~ groovy
groovyTestNG.settings.jvmArguments = "-Dsome.param=true"
~~~~ 

### Invoke dynamic

The `indy` flag toggles whether invoke dynamic support is enabled when TestNG executes your tests. This is set to `false` by default. This flag can be set like this:

~~~~ groovy
groovyTestNG.settings.indy = true
~~~~ 

### Verbosity

You can control the verbosity of the TestNG output using the `verbosity` field on the settings class. Here is an example:

~~~~ groovy
groovyTestNG.settings.verbosity = 10
~~~~ 

The value is an integer where higher values mean more verbosity.

### Report output directory

The directory that the test reports are generated in can be changed using the `reportDirectory` field on the settings class. Here is an example:

~~~~ groovy
groovyTestNG.settings.reportDirectory = Paths.get("build/reports/tests")
~~~~ 

### Dependencies

You can configure the dependencies that are included when TestNG is run using the `dependencies` field on the settings class. Here is an example:

~~~~ groovy
groovyTestNG.settings.dependencies = [
    [group: "provided", transitive: true, fetchSource: false],
    [group: "compile", transitive: true, fetchSource: false],
    [group: "runtime", transitive: true, fetchSource: false],
    [group: "test-compile", transitive: true, fetchSource: false],
    [group: "test-runtime", transitive: true, fetchSource: false]
  ]
~~~~ 

This configuration is the default. It ensures that the `provided`, `compile`, `runtime`, `test-compile`, `test-runtime` and all their transitive dependencies in any groups are included. If you want to configure a different set of groups, here are the attributes you can use for each dependency definition (the Maps inside the main list):

| name             | description                                                                               | type          | required |
|------------------|-------------------------------------------------------------------------------------------|---------------|----------|
| group            | The dependency group to include                                                           | String        | true     |
| transitive       | Determines if transitive dependencies are included or not                                 | boolean       | true     |
| fetchSource      | Determines if the source for the dependencies or downloaded or not                        | boolean       | true     |
| transitiveGroups | The transitive dependency groups to fetch. This is only used if transitive is set to true | List\<String> | false    |


## Executing tests

This plugin provides a single method to run the tests. Here is an example of calling this method:

~~~~ groovy
groovyTestNG.test()
~~~~ 

You can also control the TestNG groups that are run by passing in a set of groups to the `test` method like this:

~~~~ groovy
groovyTestNG.test(groups: ["unit"])
~~~~ 

Finally, you can run a single test using the command-line `test` switch like this:

~~~~ shell
$ latte test --test=FooBarTest
~~~~ 
