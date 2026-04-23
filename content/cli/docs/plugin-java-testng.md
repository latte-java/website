---
layout: docs
title: Java TestNG Plugin
description: The Java TestNG plugin allows you to execute your TestNG tests for your Java project.
weight: 52
plugin: true
---

## Overview

The Java TestNG plugin allows you to execute TestNG tests in a Java project. It is designed to work alongside the [Java plugin](plugin-java), which compiles and packages the sources that this plugin runs tests against. The layout, JPMS detection, and source directories described here mirror the Java plugin and share the same defaults.

**LATEST VERSION: 0.1.6**

## Loading the plugin

Here is how you load this plugin:

~~~~ groovy
javaTestNG = loadPlugin(id: "org.lattejava.plugin:java-testng:0.1.6")
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

When enabled, JaCoCo instruments the test execution via a `-javaagent` argument, writing raw coverage data to `build/jacoco.exec`. After tests complete, the plugin generates an HTML coverage report in `build/coverage-reports/`. Only the JARs in the `main` publication group are analyzed — test classes and dependencies are excluded from the report.

### Verbosity

You can control the verbosity of the TestNG output using the `verbosity` field on the settings class. Here is an example:

~~~~ groovy
javaTestNG.settings.verbosity = 10
~~~~ 

The value is an integer where higher values mean more verbosity.

### Report output directory

The directory that the test reports are output to can be changed using the `reportDirectory` field on the settings class. The default is `build/test-reports`. Here is an example:

~~~~ groovy
javaTestNG.settings.reportDirectory = Paths.get("build/reports/tests")
~~~~ 

### Dependencies

You can configure the dependencies that are included when TestNG is run using the `dependencies` field on the settings class. Here is an example:

~~~~ groovy
javaTestNG.settings.dependencies = [
    [group: "provided", transitive: true, fetchSource: false, transitiveGroups: ["provided", "compile", "runtime"]],
    [group: "compile", transitive: true, fetchSource: false, transitiveGroups: ["provided", "compile", "runtime"]],
    [group: "runtime", transitive: true, fetchSource: false, transitiveGroups: ["provided", "compile", "runtime"]],
    [group: "test-compile", transitive: true, fetchSource: false, transitiveGroups: ["provided", "compile", "runtime"]],
    [group: "test-runtime", transitive: true, fetchSource: false, transitiveGroups: ["provided", "compile", "runtime"]]
  ]
~~~~ 

This configuration is the default. It ensures that the `provided`, `compile`, `runtime`, `test-compile`, `test-runtime` and all of their transitive dependencies in any groups are included. If you want to configure a different set of groups, here are the attributes you can use for each dependency definition (which are the Maps inside the list):

| name             | description                                                                               | type          | required |
|------------------|-------------------------------------------------------------------------------------------|---------------|----------|
| group            | The dependency group to include                                                           | String        | true     |
| transitive       | Determines if transitive dependencies are included or not                                 | boolean       | true     |
| fetchSource      | Determines if the source for the dependencies or downloaded or not                        | boolean       | true     |
| transitiveGroups | The transitive dependency groups to fetch. This is only used if transitive is set to true | List\<String> | false    |


## Modules (JPMS)

The Java TestNG plugin supports running tests against projects built with the Java Platform Module System (JPMS). It works in concert with the [Java plugin](plugin-java) and mirrors the same three modes: classpath (no modules), a single module with tests patched in, and separate main and test modules.

### Auto-detection

Module mode is auto-detected from the presence of `module-info.java` files on disk. The check is performed lazily on the first plugin method call so that any overrides you make in `project.latte` are honored first.

* If `src/main/java/module-info.java` exists, `moduleBuild` is enabled.
* If `src/test/java/module-info.java` exists, `testModuleBuild` is enabled.

You can override either flag explicitly in `project.latte`:

~~~~ groovy
javaTestNG.settings.moduleBuild = true
javaTestNG.settings.testModuleBuild = false
~~~~

`testModuleBuild = true` requires `moduleBuild = true`. Enabling the test module without the main module will cause `test()` to fail with an error.

TestNG 7 and later ship with `Automatic-Module-Name: org.testng` in their JAR manifests, so TestNG resolves as an automatic module on the module path. Make sure you are using a TestNG version that supports this.

### Module mode (patched tests)

When `moduleBuild` is `true` and `testModuleBuild` is `false`, tests are executed by patching the compiled test classes into the main module. Main dependencies and the main publication JAR(s) are placed on `--module-path`; test-only dependencies (for example TestNG itself) are placed on `-classpath` (the unnamed module). The plugin automatically adds:

* `--add-modules <mainModule>` to resolve the main module,
* `--patch-module <mainModule>=<testJars>` to merge the test classes into the main module,
* `--add-reads <mainModule>=ALL-UNNAMED` so the patched tests can see classes on the classpath, and
* `--add-opens <mainModule>/<pkg>=ALL-UNNAMED` for every package in the test JARs so TestNG can reflectively invoke the test methods.

In this mode you do not write a test `module-info.java` — the tests adopt the main module's identity.

### Separate test module

When both `moduleBuild` and `testModuleBuild` are `true`, tests are executed as an independent JPMS module. The main publication JAR(s), the test publication JAR(s), and all dependencies (main and test) are placed on `--module-path`. No patching or reads-injection is performed; the test module's descriptor is authoritative.

TestNG itself is invoked via `--module org.testng/org.testng.TestNG`, and `--add-modules ALL-MODULE-PATH,<testModule>` is added so that automatic modules such as `slf4j` (which TestNG depends on) and the test module are resolved.

Because the plugin does not emit `--add-opens`, the test `module-info.java` must declare which packages TestNG is allowed to reflect into. A typical descriptor looks like this:

~~~~ java
module org.example.tests {
  requires org.example;
  requires org.testng;
  opens org.example.tests to org.testng;
}
~~~~

Every package that contains `@Test` methods must be opened to `org.testng`. If TestNG cannot access a test class at runtime, it is almost always because of a missing `opens` directive.

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

Values passed to `--test` are first matched against the simple class name and the fully-qualified class name exactly. If no exact match is found, the plugin falls back to a substring match against the class path, so `--test=FooBar` would also run `FooBarOtherTest`, `FooBarIntegrationTest`, etc.

You can optionally run only tests that failed from the last execution:

~~~~ shell
$ latte test --onlyFailed
~~~~

When a test run fails, the plugin preserves `testng-failed.xml` to a location under the system temp directory (`${java.io.tmpdir}/<project-name>/test-reports/last/testng-failed.xml`). `--onlyFailed` hands that file directly to TestNG on the next run, ignoring `groups`/`exclude` attributes. Because the preserved file lives outside the project, running `clean` does **not** invalidate it. The file persists until the OS clears its temp directory (for example, on reboot on some systems). If no preserved file exists, the plugin logs a message and returns without running any tests.

You can skip all the tests in a project using the command-line switch:

~~~~ shell
$ latte test --skipTests
~~~~

You can run only tests that have changed on the current branch or pull request:

~~~~ shell
$ latte test --onlyChanges
~~~~

The plugin determines the changed file set by trying `gh pr diff --name-only` first, and falling back to `git diff --no-merges origin..HEAD` if the `gh` CLI is unavailable or the branch is not a pull request. Uncommitted changes (`git diff -u --name-only HEAD`) are always included on top of that. The matcher only inspects `src/main/java/**/*.java` and `src/test/java/**/*Test.java` — changes to files outside those locations are ignored, and changes in upstream dependencies are not considered. For a main-source file, the corresponding `*Test.java` is run if it exists.

Two edges are worth knowing about:

* **Changes to `project.latte` are invisible to the matcher.** Bumping a dependency version, upgrading a plugin, or editing a build target does not cause any tests to run under `--onlyChanges` — no `.java` file changed. After changing `project.latte` (or anything else outside `src/main/java` / `src/test/java`), run `latte test` at least once without the flag.
* **If no tests match, the build still passes.** The plugin logs `Found [0] tests to run from changed files` and exits successfully. This matches the default behavior of Maven Surefire (`failIfNoTests=false`); it differs from Gradle, which fails when a test filter matches nothing. If you are running `--onlyChanges` in CI as your only test step, pair it with a full `latte test` run earlier in the pipeline.

You can also specify an explicit commit or commit range to diff against instead of the default branch logic:

~~~~ shell
$ latte test --onlyChanges --commitRange=HEAD~3  # Run tests affected by changes in the last 3 commits
~~~~

You can keep the generated TestNG XML file after execution:

~~~~ shell
$ latte test --keepXML
~~~~

The file is copied to `build/test/<generated-name>.xml` in the project directory, which is useful for importing the suite definition into IntelliJ or re-running it with TestNG directly.

### TestNG exit codes

The plugin interprets the TestNG process exit code as follows:

* `0` — all tests passed; the build continues.
* `2` — some tests were skipped but none failed; a message is logged and the build continues.
* `1` — one or more tests failed; `testng-results.xml` and `testng-failed.xml` are preserved to the temp directory (for `--onlyFailed`) and the build fails.
* anything else — treated as a configuration or unknown error; preserved files are written if present and the build fails.
