---
layout: docs
title: Project Layout
description: What `latte init web` creates and how the build is wired together.
weight: 20
---

## Directory structure

After running `latte init web` you'll have a layout like this:

~~~~
hello-web/
├── project.latte
├── src/
│   ├── main/
│   │   └── java/
│   │       └── Main.java
│   └── test/
│       └── java/
│           └── MainTest.java
└── web/
    └── static/
~~~~

The `web/static` directory is intended for static assets — CSS, JavaScript, images, and HTML files served by the `StaticResources` middleware. See the [Static Files](../static-files/) page for how to wire it up.

## Project file

`project.latte` is a Groovy DSL that the Latte CLI reads. The template wires up:

- A single compile dependency on `org.lattejava:web:0.1.0` (which transitively pulls in `org.lattejava:http` and `org.lattejava:jwt`)
- A test dependency on TestNG
- The `dependency`, `java`, `java-testng`, and `release-git` plugins
- Targets for `clean`, `build`, `test`, `run`, `int`, and `release`

The most relevant target for development is `run`, which uses JEP 512's `java`-based source launcher to start the server without a separate compile step:

~~~~ groovy
target(name: "run", description: "Runs the web server") {
  java.run(main: "src/main/java/Main.java")
}
~~~~

Because the `java` plugin is configured with `moduleBuild = true`, your application runs on the module path with `org.lattejava.web` and its transitive dependencies resolved as named modules. That's what makes `import module org.lattejava.web` work in `Main.java`.

## Common commands

| Command | What it does |
|---|---|
| `latte run` | Runs `Main.java` directly (fastest dev loop) |
| `latte build` | Compiles and JARs the project |
| `latte test` | Compiles, JARs, and runs TestNG |
| `latte clean` | Removes build artifacts |

For details on the Latte CLI, see the [CLI documentation](../../../cli/docs/).
