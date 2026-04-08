---
layout: docs
title: Java TestNG Plugin
description: The Java TestNG plugin allows you to execute your TestNG tests for your Java project.
weight: 52
plugin: true
---

## Overview

The Java TestNG plugin allows you to execute TestNG tests in a Java project.

**LATEST VERSION: 0.1.0**

## Loading the plugin

Here is how you load this plugin:

~~~~ groovy
javaTestNG = loadPlugin(id: "org.lattejava.plugin:java-testng:0.1.0")
~~~~ 

## Settings

The Java TestNG plugin has one required setting, which specifies the Java version used to execute the tests with. The settings for the plugin are configured via the `JavaTestNGSettings` class, which is defined on the `JavaTestNGPlugin` class via a field named `settings`. Here is an example of configuring the plugin:

~~~~ groovy
javaTestNG.settings.javaVersion = "17"
~~~~ 

This property specifies the Java version to use, but you must also configure the location of the JDK on your computer. The location of the JDK is configured in the file `~/.config/latte/plugins/org.lattejava.plugin.java.properties`.

Here's an example of the `~/.config/latte/plugins/org.lattejava.plugin.java.properties` file:

~~~~ 
1.8=/Users/me/dev/java/current8
17=/Users/me/dev/java/current17
21=/Users/me/dev/java/current21
~~~~ 

This class has additional fields that allow you to configure the command-line arguments passed to Java when it is run, the verbosity of the TestNG output, and the location of the test reports. Here are the additional settings:

### JVM arguments

You can specify additional parameters to the JVM when TestNG executes your tests using the `jvmArguments` field on the settings class. Here is an example:

~~~~ groovy
javaTestNG.settings.jvmArguments = "-Dsome.param=true"
~~~~ 

### TestNG arguments

You can specify additional TestNG arguments using the `testngArguments` field on the settings class. Here is an example:

~~~~ groovy
javaTestNG.settings.testngArguments = "-parallel methods"
~~~~

### Listeners

You can specify custom TestNG listeners using the `listeners` field on the settings class. Here is an example:

~~~~ groovy
javaTestNG.settings.listeners = ["com.mycompany.MyTestListener"]
~~~~

### Code coverage

The Java TestNG plugin has built-in support for JaCoCo code coverage. Enable it using the `codeCoverage` field on the settings class:

~~~~ groovy
javaTestNG.settings.codeCoverage = true
~~~~

When enabled, JaCoCo will instrument the test execution and produce a coverage report.

### Verbosity

You can control the verbosity of the TestNG output using the `verbosity` field on the settings class. Here is an example:

~~~~ groovy
javaTestNG.settings.verbosity = 10
~~~~ 

The value is an integer where higher values mean more verbosity.

### Report output directory

The directory that the test reports are output to can be changed using the `reportDirectory` field on the settings class. Here is an example:

~~~~ groovy
javaTestNG.settings.reportDirectory = Paths.get("build/reports/tests")
~~~~ 

### Dependencies

You can configure the dependencies that are included when TestNG is run using the `dependencies` field on the settings class. Here is an example:

~~~~ groovy
javaTestNG.settings.dependencies = [
    [group: "provided", transitive: true, fetchSource: false],
    [group: "compile", transitive: true, fetchSource: false],
    [group: "runtime", transitive: true, fetchSource: false],
    [group: "test-compile", transitive: true, fetchSource: false],
    [group: "test-runtime", transitive: true, fetchSource: false]
  ]
~~~~ 

This configuration is the default. It ensures that the `provided`, `compile`, `runtime`, `test-compile`, `test-runtime` and all of their transitive dependencies in any groups are included. If you want to configure a different set of groups, here are the attributes you can use for each dependency definition (which are the Maps inside the list):

| name             | description                                                                               | type          | required |
|------------------|-------------------------------------------------------------------------------------------|---------------|----------|
| group            | The dependency group to include                                                           | String        | true     |
| transitive       | Determines if transitive dependencies are included or not                                 | boolean       | true     |
| fetchSource      | Determines if the source for the dependencies or downloaded or not                        | boolean       | true     |
| transitiveGroups | The transitive dependency groups to fetch. This is only used if transitive is set to true | List\<String> | false    |


## Executing tests

This plugin provides a single method to run the tests. Here is an example of calling this method:

~~~~ groovy
javaTestNG.test()
~~~~ 

You can also control the TestNG groups that are run by passing in a set of groups to the `test` method like this:

~~~~ groovy
javaTestNG.test(groups: ["unit"])
~~~~ 

You can exclude specific test groups:

~~~~ groovy
javaTestNG.test(groups: ["unit"], exclude: ["slow"])
~~~~

You can run one or more tests using the command-line `test` switch like this:

~~~~ shell
$ latte test --test=FooBarTest --test=BazTest
~~~~

You can optionally provide a fully qualified test name if you have more than one test named `FooBarTest`. For example:

~~~~ shell
$ latte test --test=org.lattejava.action.FooBarTest
~~~~

You can optionally run only tests that failed from the last execution. This uses the test output from the previous run to determine which tests failed. If you perform a `clean`, this will not work. 

~~~~ shell
$ latte test --onlyFailed
~~~~

You can skip all the tests in a project using the command-line switch:

~~~~ shell
$ latte test --skipTests
~~~~

You can run only tests that have changed since the last build:

~~~~ shell
$ latte test --onlyChanges
~~~~

You can run only tests that have changed since a specific commit:

~~~~ shell
$ latte test --commitRange=HEAD~3  # Run tests affected by changes in the last 3 commits
~~~~

You can keep the generated TestNG XML file after execution:

~~~~ shell
$ latte test --keepXML             # Keep the generated TestNG XML file after execution
~~~~
