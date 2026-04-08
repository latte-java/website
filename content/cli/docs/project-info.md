---
layout: docs
title: Project Information
description: Latte project information is defined using the "project" definition in the project file.
weight: 21
---

## Overview

Latte project files must be placed at the root of the project and they should be named **project.latte**. Project files will define the project information, dependencies, workflow, and targets.

Let's start out by looking at a simple Latte project file that defines a project. Edit the file project.latte in your IDE or favorite text editor. Put this in your project file.

~~~~ groovy
project(group: "org.example", name: "my-project", version: "1.0", licenses: ["Apache-2.0"]) {
  // More info here
}
~~~~

You define your project details using the project method. This method takes a block, that allows you to define additional details about your project like dependencies. The key here is that the project requires these attributes:

* group - The group identifier for your organization (usually a reverse domain name)
* name - The name of the project
* version - A semantic version [https://semver.org](https://semver.org)
* licenses - The license(s) of your project

One of the most important details is that Latte requires that everything uses Semantic Versions. This is vitally important when Latte calculates dependencies of your project. You should read the Semantic Version specification by visiting [https://semver.org](https://semver.org).

You also need to specify one or more licenses for your project. All projects must specify at least one license. This helps Latte determine if your project is in compliance with organizational requirements about software licensing.

Latte uses [SPDX license identifiers](https://spdx.org/licenses/) for all licenses. Any valid SPDX license ID can be used. Here are some commonly used examples:

* `Apache-2.0` - Apache License 2.0
* `MIT` - MIT License
* `BSD-2-Clause` - BSD 2-Clause "Simplified" License
* `BSD-3-Clause` - BSD 3-Clause "New" or "Revised" License
* `GPL-2.0-only` - GNU General Public License v2.0 only
* `GPL-3.0-only` - GNU General Public License v3.0 only
* `LGPL-2.1-only` - GNU Lesser General Public License v2.1 only
* `LGPL-3.0-only` - GNU Lesser General Public License v3.0 only
* `EPL-1.0` - Eclipse Public License 1.0
* `EPL-2.0` - Eclipse Public License 2.0
* `MPL-2.0` - Mozilla Public License 2.0
* `CDDL-1.0` - Common Development and Distribution License 1.0

SPDX license exceptions are also supported using the `WITH` keyword. For example, `GPL-2.0-only WITH Classpath-exception-2.0`.

In addition to SPDX identifiers, Latte supports these custom license types:

* `Commercial` - Commercial license
* `Other` - Other license
* `OtherDistributableOpenSource` - Other open source license that allows the code to be freely distributed
* `OtherNonDistributableOpenSource` - Other open source license that doesn't allow the code to be freely distributed

Custom license types require that the project supply their own license text file at the root of the project. The file must follow this naming convention:

~~~~
license-<type>.txt
~~~~

So, if your project is using a Commercial license, you need to create a file named: **license-Commercial.txt** at the root of your project.
