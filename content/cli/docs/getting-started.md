---
layout: docs
title: Getting Started
description: Getting Started with Latte is simple. Just download, install and go!
weight: 10
---

## Install

Getting started with Latte is simple. You can run the install command for *nix systems or WSL2 systems:

~~~~ bash
curl -fsSL lattejava.org/cli/install | bash
~~~~

## Manual installation

If you prefer not to use the install script, you can install Latte manually by following these steps:

### Step 1

[Download the latest version of Latte](https://github.com/latte-java/cli/releases)

### Step 2

Unpack the TAR GZ file to a good location (you might need to change the version numbers below):

~~~~ bash
$ mkdir -p ~/dev/latte
$ cd ~/dev/latte
$ tar -xzvf latte-0.1.2.tar.gz
~~~~

### Step 3

Add the **bin** directory to your PATH:

~~~~ bash
$ export PATH=$PATH:~/dev/latte/bin
~~~~

### Step 4

Test it out:

~~~~ bash
$ latte --version
~~~~

For the full list of global flags, see [CLI Flags](../cli-flags/). For the built-in commands (`init`, `install`, `upgrade`), see [CLI Commands](../cli-commands/).
