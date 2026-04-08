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
