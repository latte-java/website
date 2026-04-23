---
layout: docs
title: CLI Commands
description: Reference for the top-level commands the Latte CLI provides.
weight: 12
---

## Overview

The Latte CLI provides three built-in commands plus a fallback mechanism for running build targets. The syntax is always:

~~~~ shell
$ latte [switches] <command | target>
~~~~

Any argument that is not a built-in command or a flag is treated as a build target name, and Latte looks it up in the current project's `project.latte` file. Running `latte test` invokes the `test` target; running `latte clean jar` runs `clean` then `jar`.

If a built-in command name (such as `init`) also exists as a target in the current project, the target takes precedence.

For global flags that apply to every invocation, see [CLI Flags](../cli-flags/).

## init

Initializes a new Latte project in the current directory. Prompts for the project group, name, and license, and creates a `project.latte` plus the standard source directories (`src/main/java`, `src/main/resources`, `src/test/java`, `src/test/resources`).

~~~~ shell
$ latte init
~~~~

### Options

| Option | Description |
|--------|-------------|
| `--template=<path>` | Use a custom project template file. Defaults to `${latte.home}/templates/project.latte`. |

### Prompts

| Prompt | Format |
|--------|--------|
| Group | Dot-separated identifier (e.g. `com.example`). |
| Name | Alphanumeric plus hyphens (e.g. `my-project`). |
| License | An SPDX license identifier (e.g. `Apache-2.0`). Defaults to `MIT` if left blank. See [SPDX licenses](https://spdx.org/licenses/). |

## install

Adds a dependency to your `project.latte` and downloads its artifact (plus the source JAR, when available).

~~~~ shell
$ latte install <artifact-id> [version] [dependency-group]
~~~~

If `version` is omitted, Latte queries the Latte public repository for the latest version. If `dependency-group` is omitted, the dependency is added to the `compile` group.

### Examples

Install the latest version into the default `compile` group:

~~~~ shell
$ latte install org.apache.commons:commons-collections
~~~~

Install a specific version:

~~~~ shell
$ latte install org.apache.commons:commons-collections 3.1.0
~~~~

Install into a different group:

~~~~ shell
$ latte install org.testng:testng 6.8.7 test-compile
~~~~

Repository lookup order is controlled by the project's `workflow` block. See [Workflows](../workflows/) for details on how fetch-time repository precedence works.

## upgrade

Upgrades the Latte runtime, plugins, or dependencies defined in `project.latte`.

~~~~ shell
$ latte upgrade <subcommand>
~~~~

### Subcommands

| Subcommand | Description |
|------------|-------------|
| `runtime` | Downloads the latest Latte release and replaces the `bin`, `lib`, and `templates` directories in `$latte.home`. No-op if already on the latest version. |
| `plugins` | Finds every `loadPlugin(...)` call in `project.latte`, resolves the latest version of each, and rewrites the file. |
| `dependency <artifact-id> [version]` | Upgrades a single dependency. If `version` is omitted, resolves the latest. |
| `dependencies` | Resolves the latest version of every dependency in every group and rewrites `project.latte`. |
| `all` | Runs `runtime`, `plugins`, and `dependencies` in sequence. Outside a project (no `project.latte`), only `runtime` runs; `plugins` and `dependencies` are skipped. |
| `help` | Prints usage for the upgrade subcommands. |

### Examples

Upgrade the CLI itself:

~~~~ shell
$ latte upgrade runtime
~~~~

Upgrade a single dependency to its latest version:

~~~~ shell
$ latte upgrade dependency org.apache.commons:commons-collections
~~~~

Upgrade everything — CLI, plugins, and dependencies — in one pass:

~~~~ shell
$ latte upgrade all
~~~~

## Running targets

Any argument that is not a flag is treated as a target name. If the argument is `init`, `install`, or `upgrade` and the project does not define a target of that name, Latte runs the built-in command instead (see [Overview](#overview)). Targets are defined in `project.latte` — see [Targets](../targets/).

~~~~ shell
$ latte clean
$ latte clean jar
$ latte test
~~~~

Latte runs targets in the order they are listed on the command line and resolves each target's dependencies using the graph defined in the project file. If a name on the command line is not a built-in command and is not a declared target, Latte exits with an error.

To see every target defined by the current project, use [`--listTargets`](../cli-flags/#listtargets).
