---
layout: docs
title: Output
description: You can produce output from your build using the "output" variable inside the project file.
weight: 26
---

## Overview

No matter where you are in Latte, you have access to the Output object. This object provides methods for outputting information from your build script or plugin. All methods return the Output object, so you can chain calls together.

## Methods

Each log level has two variants: one that prints without a trailing newline and one that appends a newline (the `ln` suffix).

| method | description |
| ------ | ----------- |
| `info(message, values...)` | Prints an informational message |
| `infoln(message, values...)` | Prints an informational message with a newline |
| `infoln(color, message, values...)` | Prints a colored informational message with a newline (ANSI 256 color code) |
| `warning(message, values...)` | Prints a warning message |
| `warningln(message, values...)` | Prints a warning message with a newline |
| `error(message, values...)` | Prints an error message |
| `errorln(message, values...)` | Prints an error message with a newline |
| `debug(message, values...)` | Prints a debug message (only when `--debug` is enabled) |
| `debugln(message, values...)` | Prints a debug message with a newline |
| `debug(throwable)` | Prints a stack trace as a debug message |
| `enableDebug()` | Enables debug output |
| `disableDebug()` | Disables debug output |

All message methods take the message as a String and an optional list of values that are used to fill out the message using **printf** syntax. For example:

~~~~ groovy
output.info("some info %s", "message")

// Prints "some info message"
~~~~

## Chaining

Since all methods return the Output object, you can chain calls:

~~~~ groovy
output.infoln("Building project...").debugln("Using JDK %s", jdkPath)
~~~~

## Colored output

The `infoln` method accepts an ANSI 256 color code as the first argument to print colored messages:

~~~~ groovy
output.infoln(32, "Success!")  // Prints in green
~~~~
