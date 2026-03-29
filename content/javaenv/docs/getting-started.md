---
title: "Getting Started"
layout: docs
weight: 20
---

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
