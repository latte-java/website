---
layout: docs
title: Getting Started
description: Install Latte, scaffold a web project, and run it.
weight: 10
---

## Prerequisites

You need Java 25 and the Latte CLI. If you don't already have them:

~~~~ bash
curl -fsSL https://lattejava.org/javaenv/install | bash
~~~~

~~~~ bash
javaenv install 25
~~~~

~~~~ bash
javaenv global 25
~~~~

~~~~ bash
curl -fsSL https://lattejava.org/cli/install | bash
~~~~

## Scaffold a project

Create a directory for your project and run `latte init` with the `web` template:

~~~~ bash
mkdir hello-web && cd hello-web
~~~~

~~~~ bash
latte init web
~~~~

The CLI prompts for a group, project name (defaults to the directory name), and SPDX license. When it finishes you have a fully wired Latte project that depends on `org.lattejava:web`.

## Run it

The `web` template registers a `run` target that compiles the project and starts the server with the JEP-512 `java run` mode:

~~~~ bash
latte run
~~~~

Visit `http://localhost:8080/` and you should see `Hello, world!`.

## What's in the template

The interesting file is `src/main/java/Main.java`:

~~~~ java
import module org.lattejava.http;
import module org.lattejava.web;

void main() {
  new Web()
      .get("/", (req, res) -> res.getWriter().write("Hello, world!"))
      .start(8080);
}
~~~~

A few things to notice:

- **Module imports.** `import module org.lattejava.http` and `import module org.lattejava.web` pull every exported package from those modules into scope, so `Web`, `HTTPRequest`, and `HTTPResponse` are all available without per-package imports.
- **Class-less main.** Java 25 lets the entry point live in an unnamed class with a single `void main()`. No `public class Main { ... }` required.
- **Auto-shutdown.** `start(int)` registers a JVM shutdown hook so the HTTP server closes cleanly on `Ctrl-C`. You can still wrap `Web` in a try-with-resources block when you want explicit lifetime control.

For everything else — routing, middleware, body parsing, OIDC, and static files — keep reading.
