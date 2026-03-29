---
title: "javaenv"
description: "Manage multiple Java versions on your machine"
layout: landing
github: "https://github.com/latte-java/javaenv"
---

## Prerequisites

You need a Unix-like operating system (macOS or Linux).

## Install via Homebrew

```bash
brew install latte-java/tap/javaenv
```

## Install from Source

```bash
git clone https://github.com/latte-java/javaenv.git
cd javaenv
make install
```

## Verify Installation

```bash
javaenv --version
```

## List Available Versions

```bash
javaenv list-remote
```

## Install a Java Version

```bash
javaenv install 21
```

## Set the Global Version

```bash
javaenv global 21
```

## Verify

```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello from Java " + System.getProperty("java.version"));
    }
}
```

```bash
javac Hello.java && java Hello
```
