---
layout: docs
title: Build Targets
description: Latte uses build targets defined in the project file to execute your build.
weight: 23
---

## Overview

Our Latte project file doesn't do anything yet, so let's add some targets.

~~~~ groovy
project(group: "org.example", name: "my-project", version: "1.0", licenses: ["Apache-2.0"]) {
  ...
}

target(name: "clean", description: "Cleans out the build directory") {
  ...
}

target(name: "build", description: "Compiles the project") {
  ...
}

target(name: "test", description: "Executes the projects tests", dependsOn: ["build"]) {
  ...
}
~~~~ 

This is a fairly common project file that includes targets to clean the project, build the project and run the tests. Notice that the **test** target depends on the **build** target.

## Target dependencies

Target dependencies are an ordered list. This means that Latte will ensure that dependent targets are executed in the order they are defined in the project file. For example:

~~~~ groovy
target(name: "one") {
  output.info("One")
}

target(name: "two") {
  output.info("Two")
}

target(name: "three", dependsOn: ["one", "two"]) {
  output.info("Three")
}
~~~~ 

If we run the build like this:

~~~~ shell
$ latte three
~~~~ 

Latte guarantees that the output will always be:

~~~~ 
One
Two
Three
~~~~ 
