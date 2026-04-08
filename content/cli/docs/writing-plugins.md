---
layout: docs
title: Writing Plugins
description: Latte plugins are simple to write and test.
weight: 41
plugin: true
---

## Overview

Plugins are the best part about Latte. They are simple and quick to write, simple to test, and easy to start using in your project via integration builds. Let's write a simple plugin that makes a file and puts some text in it.


## Project layout

First, we need to create the project layout. You don't need to conform to this layout, but this layout works well with the current Latte plugins.

~~~~ 
project
   |- src/main/groovy                          <- Your Groovy plugin code will go here
   |
   |- src/test/groovy                          <- Your Groovy unit tests will go here
   |
   |- project.latte                            <- Your plugin's Latte project file
   |
~~~~ 

## Groovy version

One thing to note is that Latte currently uses Groovy version `5.0.5`. This means that if you are building a plugin you **MUST** use this version of Groovy. The configuration in the Groovy plugin configuration file `~/.config/latte/plugins/org.lattejava.plugin.groovy.properties` must be set like this:

~~~~ properties
5.0=some-path-to-groovy-5.0.5
~~~~ 

## Plugin project file

Next, we need to add a Latte project file to our plugin. This project file goes in the root directory and is named `project.latte`. Here are the contents of a typical Groovy plugin's build script:

~~~~ groovy
project(group: "com.mycompany", name: "myplugin", version: "0.1.0", license: "Commercial") {
  workflow {
    standard()
  }

  dependencies {
    group(name: "provided") {
      dependency(id: "org.lattejava:latte-core:2.2.0")
      dependency(id: "org.lattejava:latte-dependency-management:2.2.0")
      dependency(id: "org.lattejava:latte-utils:2.2.0")
    }
    group(name: "test-compile", export: false) {
      dependency(id: "org.testng:testng:6.8.7")
    }
  }

  publications {
    standard()
  }
}

// Plugins
dependency = loadPlugin(id: "org.lattejava.plugin:dependency:2.0.0")
groovy = loadPlugin(id: "org.lattejava.plugin:groovy:2.0.0")
groovyTestNG = loadPlugin(id: "org.lattejava.plugin:groovy-testng:2.0.0")
release = loadPlugin(id: "org.lattejava.plugin:release-git:2.0.0")

// Plugin settings
groovy.settings.groovyVersion = "5.0"
groovy.settings.javaVersion = "17"
groovy.settings.jarManifest = [
    "Latte-Plugin-Class": "com.mycompany.MyPlugin"
]
groovyTestNG.settings.groovyVersion = "5.0"
groovyTestNG.settings.javaVersion = "17"

target(name: "clean", description: "Cleans the project") {
  groovy.clean()
}

target(name: "build", description: "Compiles the project") {
  groovy.compile()
}

target(name: "jar", description: "JARs the project", dependsOn: ["build"]) {
  groovy.jar()
}

target(name: "test", description: "Runs the project's tests", dependsOn: ["jar"]) {
  groovyTestNG.test()
}

target(name: "int", description: "Releases a local integration build of the project", dependsOn: ["test"]) {
  dependency.integrate()
}

target(name: "release", description: "Releases a full version of the project", dependsOn: ["test"]) {
  release.release()
}
~~~~ 

## The plugin class

All plugins are simple Groovy classes (or they could be written in Java, and it might be possible to write them in other JVM languages). Plugins must implement the marker interface `org.lattejava.plugin.Plugin`. Start by creating a simple Groovy class such as `src/main/groovy/com/mycompany/MyPlugin.groovy`. Add this code to the plugin class to start:

~~~~ groovy
package com.mycompany

import org.lattejava.plugin.Plugin

class MyPlugin implements Plugin {
}
~~~~ 

If you want to get some extra helper methods for your plugin, you can extend `org.lattejava.plugin.groovy.BaseGroovyPlugin`, but this isn't required.

Your plugin class must have a constructor that takes the `Project` and `Output` objects like this:

~~~~ groovy
package com.mycompany

import org.lattejava.plugin.groovy.BaseGroovyPlugin

class MyPlugin extends BaseGroovyPlugin {
  MyPlugin(project, output) {
    super(project output)
  }
}
~~~~ 

## Plugin configuration

The simplest way to configure your plugin is by creating another Groovy class in your project and setting a field in the plugin class to a new instance. Let's make a configuration class that manages the file our plugin is going to write to and add it to our plugin. Create the file `src/main/groovy/com/mycompany/MyPluginSettings.groovy` and add this code to it:

~~~~ groovy
package com.mycompany

import java.nio.file.Path

class MyPluginSettings {
  def file = Paths.get("foobar.txt")
}
~~~~ 

Now let's add the settings object to our plugin:

~~~~ groovy
package com.mycompany

import org.lattejava.plugin.groovy.BaseGroovyPlugin

class MyPlugin extends BaseGroovyPlugin {
  def settings = new MyPluginSettings()

  MyPlugin(project, output) {
    super(project output)
  }
}
~~~~ 

## Plugin methods

Lastly, you can define any number of public methods on your plugin. Each plugin method should be a separate feature of the plugin. Let's add our plugin method for creating the file and outputting some text into it:

~~~~ groovy
package com.mycompany

import java.nio.file.Files
import org.lattejava.plugin.groovy.BaseGroovyPlugin

class MyPlugin extends BaseGroovyPlugin {
  def settings = new MyPluginSettings()

  MyPlugin(project, output) {
    super(project output)
  }

  def createFile() {
    Files.write(project.directory.resolve(settings.file), "Hello World".getBytes())
  }
}
~~~~ 

## Plugin test

Now we need to test our plugin. Create the Groovy class `src/test/groovy/com/mycompany/MyPluginTest.groovy`. Add this code to this class:

~~~~ groovy
package com.mycompany

import org.lattejava.domain.Project
import org.lattejava.output.Output
import org.lattejava.output.SystemOutOutput
import org.testng.annotations.Test

import java.nio.file.Files
import java.nio.file.Paths

import static org.testng.Assert.*

class MyPluginTest {
  @Test
  def test() {
    Project project = new Project(Paths.get("build/test"), output)
    Output output = new SystemOutOutput(false)
    MyPlugin myPlugin = new MyPlugin(project, output)
    plugin.createFile()

    def bytes = Files.readAllBytes(Paths.get("build/test/foobar.txt"))
    assertEquals(new String(bytes), "Hello World")
  }
}
~~~~ 

Now we can run the tests by executing Latte from the project like this:

~~~~ 
$ latte test
~~~~ 

## Manifest entry

You might have noticed that in our `project.latte` file we added these lines of configuration:

~~~~ groovy
groovy.settings.jarManifest = [
    "Latte-Plugin-Class": "com.mycompany.MyPlugin"
]
~~~~ 

Latte uses the entry named `Latte-Plugin-Class` in the MANIFEST.MF file to load the plugin. You'll need to set this to point to your plugin class.

## Integrating

You can also run an integration build of our plugin and test it from another project. To do this, first run an integration build:

~~~~ 
$ latte int
~~~~ 

Next, go to another project and add this code to its project.latte file:

~~~~ groovy
myPlugin = loadPlugin("com.mycompany:myplugin:0.1.0-{integration}")
~~~~ 

You can now use the plugin from any project project file like this:

~~~~ groovy
target(name: "my-target") {
  myPlugin.createFile()
}
~~~~ 
