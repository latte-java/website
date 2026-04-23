---
layout: docs
title: Linter Plugin
description: The Linter plugin runs static analysis tools on your project's source code.
weight: 58
plugin: true
---

## Overview

The Linter plugin runs static analysis tools on your project's source code. Currently, it supports PMD for Java projects.

**LATEST VERSION: 0.1.5**

## Loading the plugin

Here is how you load this plugin:

~~~~ groovy
linter = loadPlugin(id: "org.lattejava.plugin:linter:0.1.5")
~~~~

## Settings

The Linter plugin has one setting for the report output directory:

### Report directory

The directory where analysis reports are written. This defaults to `build/linter-reports`:

~~~~ groovy
linter.settings.reportDirectory = Paths.get("build/linter-reports")
~~~~

## Running PMD

The `pmd` method executes PMD analysis on your source code. At minimum, you must specify the rule sets to use:

~~~~ groovy
linter.pmd(ruleSets: ["config/pmd-rules.xml"])
~~~~

You can configure the analysis with additional attributes:

~~~~ groovy
linter.pmd(
  ruleSets: ["config/pmd-rules.xml"],
  reportFormat: "html",
  reportFileName: "pmd-report",
  languageVersion: "17",
  inputPath: "src/main/java",
  minimumPriority: "MEDIUM",
  failOnViolations: true
)
~~~~

### PMD attributes

| name                       | description                             | type          | required | default            |
|----------------------------|-----------------------------------------|---------------|----------|--------------------|
| ruleSets                   | Array of ruleset file paths (required)  | List\<String> | true     | -                  |
| reportFormat               | Output format: xml, html, or text       | String        | false    | xml                |
| reportFileName             | Output report filename                  | String        | false    | pmd-report         |
| languageVersion            | Java language version                   | String        | false    | 17                 |
| inputPath                  | Source path to analyze                  | String        | false    | src/main/java      |
| sourceClassesPath          | Compiled classes path                   | String        | false    | build/classes/main |
| testClassesPath            | Test classes path                       | String        | false    | build/classes/test |
| cachePath                  | PMD cache file path                     | String        | false    | build/pmd.cache    |
| minimumPriority            | Minimum violation priority              | String        | false    | MEDIUM             |
| failOnViolations           | Whether to fail build on violations     | boolean       | false    | true               |
| reportSuppressedViolations | Include suppressed violations in report | boolean       | false    | false              |
| customRuleDependencySpecs  | Custom rule JAR dependency specs        | List\<String> | false    | []                 |
