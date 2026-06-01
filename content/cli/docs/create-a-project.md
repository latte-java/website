---
layout: docs
title: Create a Project
description: Use the init command to create a new Latte project with a project file and directory structure.
weight: 11
---

## Overview

The easiest way to start a new Latte project is with the `init` command. This command scaffolds a complete project — a `project.latte` file plus the source directories and starter files — from a template.

## Running init

Navigate to the directory you want to turn into a project and run:

~~~~ shell
$ latte init
~~~~

With no arguments, `init` uses the built-in `library` template. To select a different template, pass its name (or a path) as the first argument:

~~~~ shell
$ latte init web
~~~~

Latte then prompts you for the project's metadata:

- **Group** - The group identifier for your organization (e.g. `org.example`). Must be a dot-separated identifier.
- **Name** - The name of your project (e.g. `my-project`). Defaults to the current directory name when it is a valid identifier. May contain letters, digits, and hyphens.
- **License** - The SPDX license identifier for your project (e.g. `Apache-2.0`). Defaults to `MIT` if left blank.

When it finishes, Latte prints a confirmation:

~~~~
Created project [org.example:my-project] from template [library]
~~~~

## Choosing a template

The first positional argument selects the template directory:

| Argument                                                                              | Resolves to                                                               |
|---------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| *(omitted)*                                                                           | The built-in `library` template.                                          |
| A bare name, e.g. `web`                                                               | `$latte.home/templates/<name>` — one of the templates shipped with Latte. |
| A path (contains a `/` or `\`, starts with `~`, or is absolute), e.g. `./my-template` | That directory on disk.                                                   |

Latte ships two built-in templates:

- **`library`** - A standard Java library, configured as a [JPMS module](../plugin-java/#modules-jpms) with a separate test module.
- **`web`** - A [Latte Web](/web/) application with a `Main` class, a JTE template, and a static-file directory.

You can also create your own template directory and pass its path to `init`.

## Template variables

Inside a template, both file contents and path segments may reference these variables, which are filled in from your answers to the prompts:

| Variable         | Value                                         | Example              |
|------------------|-----------------------------------------------|----------------------|
| `${group}`       | The group you entered                         | `org.example`        |
| `${name}`        | The project name you entered                  | `my-lib`             |
| `${license}`     | The SPDX license you entered                  | `Apache-2.0`         |
| `${nameId}`      | The name with hyphens replaced by underscores | `my_lib`             |
| `${package}`     | `${group}.${nameId}`                          | `org.example.my_lib` |
| `${packagePath}` | `${package}` with dots replaced by slashes    | `org/example/my_lib` |

For example, a template file at `src/main/java/${packagePath}/Placeholder.java` is written to `src/main/java/org/example/my_lib/Placeholder.java`.

## What the `library` template generates

~~~~
my-lib/
├── .gitignore
├── project.latte
└── src/
    ├── main/
    │   ├── java/
    │   │   ├── module-info.java
    │   │   └── org/example/my_lib/Placeholder.java
    │   └── resources/
    └── test/
        ├── java/
        │   ├── module-info.java
        │   └── org/example/my_lib/tests/PlaceholderTest.java
        └── resources/
~~~~

Generated projects are JPMS modules by default. The main module exports its package, and the tests live in a **separate test module** in their own package (e.g. `org.example.my_lib.tests`). The generated `src/test/java/module-info.java` requires the main module and opens the test package to TestNG. See the [Modules (JPMS)](../plugin-java/#modules-jpms) section of the Java Plugin docs for the full picture.

The generated `project.latte` is a complete, runnable build — `standard()` workflow, a `latte()` [publish workflow](../publishing/), the `dependency`/`idea`/`java`/`java-testng`/`release-git` plugins (pinned to their current versions), and `clean`/`build`/`test`/`int`/`release`/`idea`/`print-dependency-tree` targets:

~~~~ groovy
project(group: "org.example", name: "my-lib", version: "0.1.0", licenses: ["Apache-2.0"]) {
  workflow {
    standard()
  }
  publishWorkflow {
    latte()
  }

  dependencies {
    group(name: "compile") {
    }
    group(name: "test-compile", export: false) {
      dependency(id: "org.testng:testng:7.10.2")
    }
  }

  publications {
    standard()
  }
}
~~~~

## Conflicts

`init` never overwrites existing files. Before writing anything, it checks every target path; if one already exists it fails with an error such as `[project.latte] already exists` and writes nothing.

From here you can start adding dependencies, plugins, and targets to your project file. See [Project Files](../project-files/) for a full guide.
