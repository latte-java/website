---
layout: docs
title: Plugins
description: Plugins provide re-usable logic to your project files and are simple to configure and use.
weight: 25
---

## Overview

At this point, you should have a functional project file that might include targets, dependencies, and a workflow. However, it doesn't do very much. You could start writing your build using Groovy code directly inside the targets. This would require quite a bit of work though. Instead of writing this code yourself, you can leverage plugins to fill out your build.

We'll assume that your project is a Java project and that you need to compile Java source files. To accomplish this task, we will include the Java plugin and assign it to a variable. This will look like this:

~~~~ groovy
java = loadPlugin(id: "org.lattejava.plugin:java:0.1.0")
~~~~ 

This code has to be put after the project and workflow definition because Latte uses the workflow to download and instantiate the plugin. The key to Latte's plugin mechanism is that Plugins are simply Groovy objects. They aren't abstracted in any way. That means after this line of code executes, the variable **java** will be an instance of the class `org.lattejava.plugin.java.JavaPlugin`. Any public methods or fields on that instance can be invoked to perform parts of your build.

For the Java plugin, you need to define the version of the JDK to compile with (in case you have multiple JDKs installed or need to compile with a JDK besides the one Latte requires to run).

~~~~ groovy
java = loadPlugin(id: "org.lattejava.plugin:java:0.1.0")
java.settings.javaVersion = "25"
~~~~ 

Finally, you need to create a special configuration file for the Java plugin that tells Latte the location of the JDK. Create the file `~/.config/latte/plugins/org.lattejava.plugin.java.properties` and put the full path to the JDK base directory in it like this:

~~~~ 
25=/some-path-to-java/jdk25
~~~~ 

Now that you have loaded and configured the Java plugin, you can update your targets to use it. Plugins are simple Java objects and the public methods define their features. The public fields are often used to configure the plugin. Here's the project file again with the Java plugin calls added:

~~~~ groovy
project(group: "org.example", name: "my-project", version: "1.0", licenses: ["Apache-2.0"]) {
  workflow {
    standard()
  }

  dependencies {
    group(name: "compile") {
      dependency(id: "org.apache.commons:commons-collections:3.1.0")
    }
  }
}

java = loadPlugin(id: "org.lattejava.plugin:java:0.1.0")
java.settings.javaVersion = "25"

target(name: "clean", description: "Cleans out the build directory") {
  java.clean()
}

target(name: "build", description: "Compiles the project") {
  java.compile()
}

target(name: "test", description: "Executes the projects tests", dependsOn: ["build"]) {
  ...
}
~~~~ 

Most plugins provide detailed instructions on how to configure them at runtime. If you run Latte without the proper configuration, you should get a nice error message that tells you exactly how to configure the plugin to work properly.
