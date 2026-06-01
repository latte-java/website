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

When the server starts it logs a clickable URL — `Web application is available at [http://localhost:8080]`. Visit it and you should see the page rendered from the template (`Welcome to Latte Java!`).

## What's in the template

The interesting file is `src/main/java/<package>/Main.java`. It serves static files and renders a [JTE template](templates/) for the home page:

~~~~ java
import module java.base;
import module org.lattejava.http;
import module org.lattejava.web;

public class Main {
  public static final int PORT = 8080;
  private final JTETemplates templates = new JTETemplates(Path.of("web/templates"), Path.of("build"));
  public final Web web = new Web();

  public void main() {
    web.install(SecurityHeaders.defaults())
       .baseDir(Path.of("web"))
       .files("/static")
       .get("/", templates::html)
       .start(PORT);
  }
}
~~~~

It comes with `web/templates/index.jte` (the rendered home page) and a `web/static/` directory for assets.

A few things to notice:

- **Module imports.** `import module org.lattejava.http` and `import module org.lattejava.web` pull every exported package from those modules into scope, so `Web`, `HTTPRequest`, `HTTPResponse`, `SecurityHeaders`, and `JTETemplates` are all available without per-package imports.
- **Templates.** `templates::html` is a `Handler` reference — `JTETemplates` maps the request path to a template (`/` → `index.jte`) and writes the rendered HTML. See [Templates](templates/).
- **Security by default.** `SecurityHeaders.defaults()` applies strict response headers to every response.
- **Auto-shutdown.** `start(int)` registers a JVM shutdown hook so the HTTP server closes cleanly on `Ctrl-C`. You can register cleanup work with `web.addShutdownTask(Runnable)`, or wrap `Web` in a try-with-resources block for explicit lifetime control.

For everything else — routing, middleware, body parsing, OIDC, and static files — keep reading.
