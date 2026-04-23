---
layout: docs
title: Java Plugin
description: The Java plugin allows you to compile and JAR your project's Java source code.
weight: 50
plugin: true
---

## Overview

The Java plugin allows you to build Java projects. This plugin includes methods for compiling, JARring, and cleaning a Java project.

**LATEST VERSION: 0.1.0**

## Loading the plugin

Here is how you load this plugin:

~~~~ groovy
java = loadPlugin(id: "org.lattejava.plugin:java:0.1.0")
~~~~ 

## Layout

The default layout for a Java project is:

~~~~ 
project
   |- src/main
   |       |- java        <- Your main Java source files go here
   |       |- resources   <- Your main resources (files that are needed at runtime and will be placed into the JAR)
   |
   |- src/test
   |       |- java        <- Your test Java source files go here
   |       |- resources   <- Your test resources (files that are needed at test-time and will be placed into the test JAR)
   |
   |- build
   |    |- classes
   |    |     |- main     <- The main Java source files are compiled into this directory and the main resources are copied into this directory
   |    |     |- test     <- The test Java source files are compiled into this directory and the test resources are copied into this directory
   |    |
   |    |- jars           <- The JAR files that contain your compiled Java files are created here
~~~~ 

You can change the layout via the `JavaLayout` class. The JavaPlugin class has a member field named `layout` that is the main instance of this class. Here's an example:

~~~~ 
java.layout.mainSourceDirectory = Paths.get("source/main/java")
java.layout.testSourceDirectory = Paths.get("source/test/java")
~~~~ 

This changes the location of the source files from `src/main/java` and `src/test/java` to `source/main/java` and `source/test/java`. Also, you can change these locations using the same method in the example above:

* `docDirectory`
* `jarOutputDirectory`
* `mainResourceDirectory`
* `mainBuildDirectory`
* `testResourceDirectory`
* `testBuildDirectory`

## Settings

The Java plugin has one required setting, which defines the Java version to compile with. The settings for the plugin are configured via the `JavaSettings` class, which is defined on the `JavaPlugin` class via a field named `settings`. Here is an example of configuring the plugin:

~~~~ groovy
java.settings.javaVersion = "17"
~~~~ 

This property specifies the version of Java to use, but you must also configure the location of the JDK on your computer. The location of the JDK is configured in the file `~/.config/latte/plugins/org.lattejava.plugin.java.properties`. Here's an example of the `~/.config/latte/plugins/org.lattejava.plugin.java.properties` file:

~~~~ 
1.8=/Users/me/dev/java/current8
17=/Users/me/dev/java/current17
21=/Users/me/dev/java/current21
~~~~ 

This class has additional fields that allow you to configure the command-line arguments passed to the javac compiler and the dependencies to include during compilation. Here are the additional parameters:

### Compiler arguments

You can specify additional parameters to the Java compiler using the `compilerArguments` field on the settings class. Here is an example:

~~~~ groovy
java.settings.compilerArguments = "-g"
~~~~ 

This is the default setting for this field, and it specifies that debug information should be included by the Java compiler.

### Doc arguments

You can specify additional parameters to the Javadoc generator using the `docArguments` field on the settings class. Here is an example:

~~~~ groovy
java.settings.docArguments = "-footer '<span>Copyright 2014 My Company</span>'"
~~~~ 

### JVM arguments

You can specify additional JVM arguments when running Java tools using the `jvmArguments` field on the settings class. Here is an example:

~~~~ groovy
java.settings.jvmArguments = "-Xmx512m"
~~~~

### Additional library directories

In some cases, you don't have access to a Latte repository that contains certain JAR files that you need for compilation. The best practice is to add the missing JAR files to a Latte repository and use Latte's dependency management system to manage them. However, some third party JAR files might have unknown transitive dependencies. Rather than trying to figure out all of these complex dependencies, Latte provides you with a simple way to include additional JARs in the compile time classpath. You can specify one or more directories that contain JAR files, and Latte will include all the JAR files in the classpath. Here is an example:

~~~~ groovy
java.settings.libraryDirectories = ["lib"]
~~~~ 

### Compile dependencies

You can change the dependencies that are included in the classpath during compilation. This is an advanced setting and should be only used in special cases. This setting is controlled by the `mainDependencies` and `testDependencies` fields. These fields are a List of Maps. Each Map defines a dependency group to include, whether transitive dependencies are included, and what transitive dependency groups are also included. Here is an example:

~~~~ groovy
java.settings.mainDependencies = [
    [group: "compile", transitive: false, fetchSource: false],
    [group: "provided", transitive: false, fetchSource: false]
  ]
java.settings.testDependencies =  [
    [group: "compile", transitive: false, fetchSource: false],
    [group: "test-compile", transitive: false, fetchSource: false],
    [group: "provided", transitive: false, fetchSource: false]
  ]

~~~~ 

This defines that the project's `compile` and `provided` dependency groups should be included when the main classes are compiled but transitive dependencies are not included. Likewise, the `compile`, `test-compile` and `provided` dependency groups are included when the test classes are compiled, but transitive dependencies are not. This is the default setting and should be used in most projects, but you can modify these settings if necessary. If you choose to change these definitions, the attributes you can specify for dependency definitions are:

| name             | description                                                                               | type          | required |
|------------------|-------------------------------------------------------------------------------------------|---------------|----------|
| group            | The dependency group to include                                                           | String        | true     |
| transitive       | Determines if transitive dependencies are included or not                                 | boolean       | true     |
| fetchSource      | Determines if the source for the dependencies or downloaded or not                        | boolean       | true     |
| transitiveGroups | The transitive dependency groups to fetch. This is only used if transitive is set to true | List\<String> | false    |

### JAR manifest

You can control the manifest attributes for the JAR file that the Java plugin builds for your project. This plugin ensures that these manifest attributes are always set:

* Manifest-Version - this is set to `1.0`
* Implementation-Version - this is set to the project version
* Implementation-Vendor - this is set to the group and name of the project concatenated together with a dot separator
* Specification-Version - this is set to the project version
* Specification-Vendor - this is set to the group and name of the project concatenated together with a dot separator

Here is an example of setting additional manifest attributes or overriding the defaults listed above:

~~~~ groovy
java.settings.jarManifest = [
    "Implementation-Vendor": "My Company",
    "Specification-Vendor": "My Company",
    "Main-Class": "com.mycompany.Main"
]
~~~~ 

## Modules (JPMS)

The Java plugin has full support for the Java Platform Module System (JPMS). It can build a project as a classic classpath-based project, as a single module, or as two separate modules (one for main, one for tests).

### Auto-detection

Module mode is auto-detected from the presence of `module-info.java` files on disk. The check is performed lazily on the first plugin method call so that any overrides you make in `project.latte` are honored first.

* If `src/main/java/module-info.java` exists, `moduleBuild` is enabled.
* If `src/test/java/module-info.java` exists, `testModuleBuild` is enabled.

You don't normally need to configure anything — just add the appropriate `module-info.java` files and the plugin switches modes automatically. If you want to force a behavior (for example, to temporarily disable module mode for an incremental migration), you can set the flags explicitly:

~~~~ groovy
java.settings.moduleBuild = true
java.settings.testModuleBuild = false
~~~~

`testModuleBuild = true` requires `moduleBuild = true`. If you enable the test module without the main module, `compileTest()` will fail with an error.

### Module mode (patched tests)

When `moduleBuild` is `true` and `testModuleBuild` is `false`, the main source tree is compiled as a JPMS module (Latte uses `--module-path` for main compilation, and `javadoc` is invoked with `--module <moduleName>`). Test sources are compiled using `--patch-module`, which injects them into the main module so they can access package-private classes. Test-only dependencies (such as TestNG) remain on the classic `-classpath` (the unnamed module), and `--add-reads <mainModule>=ALL-UNNAMED` is added automatically so the patched test classes can see them.

The project layout for this mode is:

~~~~
project
   |- src/main/java
   |       |- module-info.java          <- Declares the main module
   |       |- org/example/...           <- Main Java sources
   |
   |- src/test/java
   |       |- org/example/...           <- Test Java sources (patched into the main module)
~~~~

There is no test `module-info.java` in this layout. The test classes inherit the main module's identity and therefore have full access to its internals.

### Separate test module

When both `moduleBuild` and `testModuleBuild` are `true`, the test tree is compiled as an independent JPMS module with its own `module-info.java`. The test module must declare `requires` for the main module and for any test-only dependencies it uses. No `--patch-module` or `--add-reads` is used — the test module's descriptor is authoritative.

The project layout for this mode is:

~~~~
project
   |- src/main/java
   |       |- module-info.java          <- Main module descriptor
   |       |- org/example/...
   |
   |- src/test/java
   |       |- module-info.java          <- Test module descriptor (requires main + testng)
   |       |- org/example/tests/...
~~~~

A typical test `module-info.java` looks like this:

~~~~ java
module org.example.tests {
  requires org.example;
  requires org.testng;
  opens org.example.tests to org.testng;
}
~~~~

The `opens ... to org.testng` directive is required so that TestNG can reflectively discover and invoke the test methods when the tests are executed by the Java TestNG plugin.

During test compilation, the main build directory is placed on `--module-path` so the test module can resolve `requires <mainModule>`. The test build directory is intentionally kept off the module path during compilation to avoid the in-progress module conflicting with the module being compiled.

#### Test packages must differ from main packages

JPMS forbids two modules from declaring the same package. Because the test module is a *separate* module, its classes have to live in different packages from the main module. A common convention is to append `.tests` to the package names — for example, main code in `org.example.foo` with tests in `org.example.foo.tests`.

If you try to put test classes in a package that already exists in the main module, the compiler will fail with:

~~~~
package exists in another module: org.example.foo
~~~~

Each test package must also be `opens`ed to the test framework so that reflection-driven test discovery can see the test methods. A minimal test `module-info.java` for a main module named `org.example.foo` looks like this:

~~~~ java
module org.example.foo.tests {
  requires org.example.foo;      // access the exported public API
  requires org.testng;           // test framework
  requires org.easymock;         // if using EasyMock
  // requires static org.slf4j;  // compile-time-only deps

  opens org.example.foo.tests to org.testng;
}
~~~~

Test classes then go under `src/test/java/org/example/foo/tests/`.

#### Automatic module names

Java 9+ JARs without a `module-info.class` are usable as *automatic modules*. Their module name is derived from (in priority order):

1. The `Automatic-Module-Name` manifest attribute, if present.
2. Otherwise, from the JAR filename, by stripping the version and replacing non-alphanumeric characters with `.`.

Common test-framework module names:

| Library | Automatic module name |
|---------|-----------------------|
| TestNG 7+ | `org.testng` (from manifest) |
| EasyMock 5+ | `org.easymock` (from manifest) |
| slf4j-api | `org.slf4j` (from manifest) |
| jcommander | `jcommander` (derived from filename, not stable across rebrandings) |

If you see `module not found: <name>` at compile time, inspect the JAR to learn what name JPMS has assigned it:

~~~~ shell
$ jar --file path/to/library.jar --describe-module
# or
$ unzip -p path/to/library.jar META-INF/MANIFEST.MF | grep -i module
~~~~

#### Resource loading

Test classes often load resources via `ClassLoader.getResourceAsStream(...)`. Under JPMS this only works if the resource's package is `opens`ed or `exports`ed by the owning module. Two common cases:

* A resource at `src/test/resources/config.properties` ends up in the test JAR's *root* — no package — and is always accessible.
* A resource at `src/test/resources/org/example/foo/tests/fixture.json` is in the `org.example.foo.tests` package, which is already `opens`ed to the test framework for test discovery, so no extra configuration is needed.

Mismatches surface as `InputStream` being `null`, not as exceptions. When tests that previously passed start reading resources as `null`, double-check the `opens` directives.

#### Runtime execution (java-testng plugin)

When `testModuleBuild` is `true`, the `java-testng` plugin runs tests with:

~~~~
java --module-path <all-deps-and-jars> \
     --add-modules ALL-MODULE-PATH,<testModuleName> \
     --module org.testng/org.testng.TestNG ...
~~~~

`ALL-MODULE-PATH` forces resolution of every JAR on the module path. This is necessary because TestNG's transitive dependencies (slf4j-api, jcommander, etc.) are automatic modules that aren't explicitly required by anything; without `ALL-MODULE-PATH` they would not be resolved and TestNG would fail at startup with `NoClassDefFoundError`.

The `--module org.testng/org.testng.TestNG` entry point depends on TestNG shipping `Automatic-Module-Name: org.testng`, which TestNG 7.0+ does. If you pin to an older TestNG, separate-test-module mode will not work — stay on the patched-tests mode, or upgrade TestNG.

### When to use separate test modules

Choose separate test modules when you want tests to be limited to the main module's *exported* API. Tests physically cannot access internals, so accidentally relying on non-exported packages fails at compile time. This is a strong guarantee that your tests exercise the same surface your consumers see.

Stay in patched-tests mode when:

* You need to test package-private classes or non-exported packages.
* You depend on legacy JARs whose derived automatic-module names are not stable.
* Your project's main and test package trees deliberately overlap.

Patched-tests mode is still the default when only `src/main/java/module-info.java` exists. It is not deprecated.

### JARs and Javadoc

When `moduleBuild` is enabled, the `module-info.class` produced by the compiler is included in the main JAR created by `jar()`. When `testModuleBuild` is enabled, the test JAR similarly contains its own `module-info.class`. The `document()` method automatically switches to module-aware Javadoc generation by passing `--module <moduleName>` to `javadoc`.

## Compiling

The `compile` method on the Java plugin allows you to compile the Java source files. This compiles both the main and test source files in the project. It also copies the main and test resource files to the build directory. Here's an example of calling this method:

~~~~ groovy
java.compile()
~~~~ 

You can also compile the main or test source files separately using the `compileMain` and `compileTest` methods like this:

~~~~ groovy
java.compileMain()
java.compileTest()
~~~~ 

## Documenting

The `document` method on the Java plugin generates Javadoc for your project. The output is placed in the project's configured doc directory as outlined above. Here is an example:

~~~~ groovy
java.document()
~~~~

## JARring

The `jar` method on the Java plugin allows you to build the project JAR files. This produces three JAR files, the main JAR, the test JAR, and a sources JAR file. The main JAR contains all the files from the `build/classes/main` directory. The test JAR file contains all the files from the `build/classes/test` directory. And the source JAR file contains the source files from the `src/java/main` directory. Here is an example of using this method.

~~~~ groovy
java.jar()
~~~~ 

## Cleaning

The `clean` method on the Java plugin deletes the entire `build` directory. Here's an example of using this method:

~~~~ groovy
java.clean()
~~~~

## Advanced usage

The Java plugin provides additional methods for advanced use cases:

### Main classpath

You can retrieve the main classpath for the project using the `getMainClasspath` method. This returns a classpath string that includes all the `compile`, `runtime`, and `provided` dependencies:

~~~~ groovy
String cp = java.getMainClasspath()
~~~~

### JarJar

The `jarjar` method allows you to shade and repackage dependencies using `JarJar`. This is useful when you need to bundle dependencies into your JAR with relocated packages to avoid conflicts:

~~~~ groovy
java.jarjar(dependencyGroup: "compile") {
  rule(from: "org.example.**", to: "com.mycompany.shaded.org.example.@1")
}
~~~~

The `dependencyGroup` attribute specifies which dependency group to shade. Inside the closure, use the `rule` method to define package relocation rules with `from` and `to` attributes.
