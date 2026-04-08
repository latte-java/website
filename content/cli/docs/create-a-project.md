---
layout: docs
title: Create a Project
description: Use the init command to create a new Latte project with a project file and directory structure.
weight: 11
---

## Overview

The easiest way to start a new Latte project is with the `init` command. This command creates a project file and the standard directory structure for you.

## Running init

Navigate to an empty directory and run:

~~~~ shell
$ latte init
~~~~

Latte will prompt you for the following information:

- **Group** - The group identifier for your organization (e.g. `org.example`)
- **Name** - The name of your project (e.g. `my-project`)
- **License** - The SPDX license identifier for your project (e.g. `Apache-2.0`)

## What it generates

After answering the prompts, Latte creates the following structure:

~~~~
my-project/
├── project.latte
├── src/
│   ├── main/
│   │   ├── java/
│   │   └── resources/
│   └── test/
│       ├── java/
│       └── resources/
~~~~

The generated `project.latte` file will contain your project definition with the values you provided:

~~~~ groovy
project(group: "org.example", name: "my-project", version: "0.1.0", licenses: ["Apache-2.0"]) {
  workflow {
    standard()
  }
}
~~~~

From here you can start adding dependencies, plugins, and targets to your project file. See [Project Files](../project-files/) for a full guide.
