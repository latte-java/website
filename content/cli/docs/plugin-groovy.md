---
layout: docs
title: Groovy Plugin
description: The Groovy plugin allows you to compile and JAR your Groovy source code.
weight: 51
plugin: true
---

## Overview

The Groovy plugin allows you to build Groovy projects. This plugin includes methods for compiling, jarring, and cleaning a Groovy project.

**LATEST VERSION: 0.1.5**


## Loading the plugin

Here is how you load this plugin:

~~~~ groovy
groovy = loadPlugin(id: "org.lattejava.plugin:groovy:0.1.5")
~~~~ 

## Layout

The default layout for a Groovy project is:

~~~~ 
project
   |- src/main
   |       |- groovy      <- Your main Groovy source files go here
   |       |- resources   <- Your main resources (files that are needed at runtime and will be placed into the JAR)
   |
   |- src/test
   |       |- groovy      <- Your test Groovy source files go here
   |       |- resources   <- Your test resources (files that are needed at test-time and will be placed into the test JAR)
   |
   |- build
   |    |- classes
   |    |     |- main     <- The main Groovy source files are compiled into this directory and the main resources are copied into this directory
   |    |     |- test     <- The test Groovy source files are compiled into this directory and the test resources are copied into this directory
   |    |
   |    |- jars           <- The JAR files that contain your compiled Groovy files are created here
~~~~ 

You can change the layout via the **GroovyLayout** class. The GroovyPlugin class has a member field named **layout** that is the main instance of this class. Here's an example:

~~~~ 
groovy.layout.mainSourceDirectory = Paths.get("source/main/groovy")
groovy.layout.testSourceDirectory = Paths.get("source/test/groovy")
~~~~ 

This changes the location of the source files from **src/main/groovy** and **src/test/groovy** to **source/main/groovy** and **source/test/groovy**. You can also change these locations as well using the same method in the example above:

* docDirectory
* jarOutputDirectory
* mainResourceDirectory
* mainBuildDirectory
* testResourceDirectory
* testBuildDirectory


## Settings

The Groovy plugin requires a couple of configuration settings including the Groovy compiler version and the Java version to compile with. The settings for the plugin are configured via the **GroovySettings** class. The **GroovyPlugin** class has a field named **settings** that is the main instance of this class. Here is an example of configuring the plugin:

~~~~ groovy
groovy.settings.groovyVersion = "5.0"
groovy.settings.javaVersion = "17"
~~~~ 

These two properties specify the versions to use, but you must also configure the location of the GDK and JDK on your computer. These locations are configured in two configuration files in your home directory. These files are:

* ~/.config/latte/plugins/org.lattejava.plugin.java.properties
* ~/.config/latte/plugins/org.lattejava.plugin.groovy.properties

The first file is used to define the location of the JDKs on your computer. Here's an example of the **~/.config/latte/plugins/org.lattejava.plugin.java.properties** file:

~~~~ 
17=/Users/me/dev/java/current17
21=/Users/me/dev/java/current21
25=/Users/me/dev/java/current25
~~~~ 

Here's an example of the **~/.config/latte/plugins/org.lattejava.plugin.groovy.properties** file:

~~~~ 
4.0=/Users/me/dev/groovy/groovy-4.0.22
5.0=/Users/me/dev/groovy/groovy-5.0.5
~~~~ 

This class has additional fields that allow you to configure the command-line arguments passed to the groovyc compiler, whether or not to use invokedynamic support, and the dependencies to include during compilation. Here are the additional parameters:

### Compiler arguments

You can specify additional parameters to the Groovy compiler using the **compilerArguments** field on the settings class. Here is an example:

~~~~ groovy
groovy.settings.compilerArguments = "-j"
~~~~ 

This will enable joint compilation in the Groovy compiler.

### Doc arguments

You can specify additional parameters to the Groovy doc generator using the **docArguments** field on the settings class. Here is an example:

~~~~ groovy
groovy.settings.docArguments = "-footer '<span>Copyright 2014 My Company</span>'"
~~~~ 

### JVM arguments

You can specify additional JVM arguments passed to the Groovy compiler using the **jvmArguments** field on the settings class. Here is an example:

~~~~ groovy
groovy.settings.jvmArguments = "-Xmx512m"
~~~~

### Invoke dynamic

The **indy** flag toggles whether or not your Groovy classes are compiled with invoke dynamic support or not. This is set to **false** by default. This flag can be set like this:

~~~~ groovy
groovy.settings.indy = true
~~~~ 

### Compile dependencies

You can change the dependencies that are included in the classpath during compilation. This is an advanced setting and should be only used in very special cases. This setting is controlled by the **mainDependencies** and **testDependencies** fields. These fields are a List of Maps. Each Map defines a dependency group to include and also whether or not transitive dependencies are included and what transitive dependency groups are also included. Here is an example:

~~~~ groovy
groovy.settings.mainDependencies = [
    [group: "compile", transitive: false, fetchSource: false],
    [group: "provided", transitive: false, fetchSource: false]
  ]
groovy.settings.testDependencies =  [
    [group: "compile", transitive: false, fetchSource: false],
    [group: "test-compile", transitive: false, fetchSource: false],
    [group: "provided", transitive: false, fetchSource: false]
  ]

~~~~ 

This defines that the project's **compile** and **provided** dependency groups should be included when the main classes are compiled but transitive dependencies are not included. Likewise, this includes the **compile**, **test-compile** and **provided** dependency groups are included when the test classes are compiled, but transitive dependencies are not. This is the default setting and should be used in almost every project, but you can modify these settings if necessary. If you choose to change these definitions, the attributes you can specify for dependency definitions are:

| name | description | type | required |
| ---- | ----------- | ---- | -------- |
| group | The dependency group to include | String | true |
| transitive | Determines if transitive dependencies are included or not | boolean | true |
| fetchSource | Determines if the source for the dependencies or downloaded or not | boolean | true |
| transitiveGroups | The transitive dependency groups to fetch. This is only used if transitive is set to true | List\<String> | false |

### JAR manifest

You can control the manifest attributes for the JAR file that the Groovy plugin builds for your project. This plugin ensures that these manifest attributes are always set:

* Manifest-Version - this is set to **1.0**
* Implementation-Version - this is set to the project version
* Implementation-Vendor - this is set to the group and name of the project concatenated together with a dot separator
* Specification-Version - this is set to the project version
* Specification-Vendor - this is set to the group and name of the project concatenated together with a dot separator

Here is an example of setting additional manifest attributes or overriding the defaults listed above:

~~~~ groovy
groovy.settings.jarManifest = [
    "Implementation-Vendor": "My Company",
    "Specification-Vendor": "My Company",
    "Main-Class": "com.mycompany.Main"
  ]
~~~~ 

## Compiling

The **compile** method on the Groovy plugin allows you to compile the Groovy source files. This compiles both the main and test source files in the project and it also copies the main and test resource files to the build directory. Here's an example of calling this method:

~~~~ groovy
groovy.compile()
~~~~ 

You can also compile the main or test source files separately using the **compileMain** and **compileTest** methods like this:

~~~~ groovy
groovy.compileMain()
groovy.compileTest()
~~~~ 


## Jarring

The **jar** method on the Groovy plugin allows you to build the project Jar files. This produces two Jar files, the main Jar and the test Jar. The main Jar contains all of the files from the **build/classes/main** directory and the test Jar file contains all of the files from the **build/classes/test** directory. Here is an example of using this method.

~~~~ groovy
groovy.jar()
~~~~ 


## Cleaning

The **clean** method on the Groovy plugin simply deletes the **build** directory. Here's an example of using this method:

~~~~ groovy
groovy.clean()
~~~~ 


## Advanced usage

The Groovy plugin also defines a few more advanced methods for compiling Groovy files in any directory to any directory or copying resources from any directory to any directory.
