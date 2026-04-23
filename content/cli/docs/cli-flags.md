---
layout: docs
title: CLI Flags
description: Reference for the global flags that apply to every invocation of the Latte CLI.
weight: 13
---

## Overview

Global flags apply to every invocation of `latte`, regardless of which command or target you are running. Flags may appear anywhere on the command line — before a command, after a target, or interleaved.

Any flag that starts with `--` and is not one of the flags listed below is treated as a user-defined switch. User-defined switches are made available to your `project.latte` file at runtime, allowing you to pass ad-hoc values into a build (for example, `latte --env=prod deploy`).

For the list of commands those flags apply to, see [CLI Commands](../cli-commands/).

## Flags

| Flag | Purpose |
|------|---------|
| `--debug` | Enables debug output and prints full stack traces on failure. |
| `--help` | Prints the usage summary and exits. |
| `--listTargets` | Prints every target defined in the current project's `project.latte` and exits. |
| `--noColor` | Disables ANSI color codes in all output. |
| `--version` | Prints the Latte version and exits. |

### --debug

Turns on debug-level logging and, when a build fails, prints the full stack trace of the underlying exception rather than just the message. Useful when a failure's surface error doesn't explain what happened.

### --help

Prints a short usage summary covering the built-in commands and flags, then exits. Run `latte --help` any time you want a refresher without visiting the docs.

### --listTargets

Lists every target defined in the current project's `project.latte` and exits without running anything. This is the fastest way to see what a project can do:

~~~~ shell
$ latte --listTargets
~~~~

### --noColor

Disables ANSI color codes in Latte's output. Useful when piping output to a file, capturing logs from CI, or running in a terminal that does not render color escapes.

### --version

Prints the installed Latte version and exits. Use this after installing or upgrading to confirm which version is on your PATH:

~~~~ shell
$ latte --version
~~~~

## User-defined switches

Any flag beginning with `--` that is not listed above is captured as a user-defined switch and made available to your `project.latte`. Switches support an optional `=value` syntax; flags without a value are treated as boolean.

~~~~ shell
$ latte --env=prod --skipTests deploy
~~~~

How your project reads these switches is up to the project file — see [Variables](../variables/) for the mechanisms available.

## Debugging a failed build

When a build fails and the error message alone isn't enough, combine `--debug` and `--noColor` to get a clean, complete log suitable for pasting into a bug report:

~~~~ shell
$ latte --debug --noColor test 2>&1 | tee latte-debug.log
~~~~
