---
layout: docs
title: Project Layout
description: What `latte init web` creates and how the build is wired together.
weight: 20
---

## Directory structure

After running `latte init web` (with group `org.example`, name `hello-web`) you'll have a layout like this:

~~~~
hello-web/
├── .gitignore
├── project.latte
├── src/
│   ├── main/
│   │   └── java/
│   │       ├── module-info.java
│   │       └── org/example/hello_web/Main.java
│   └── test/
│       └── java/
│           ├── module-info.java
│           └── org/example/hello_web/tests/MainTest.java
└── web/
    ├── static/
    └── templates/
        └── index.jte
~~~~

The project is a JPMS module with a [separate test module](../../../cli/docs/plugin-java/#modules-jpms). The `web/static` directory holds static assets (CSS, JavaScript, images) served by the `StaticResources` middleware, and `web/templates` holds the [JTE templates](../templates/) the app renders. See [Static Files](../static-files/) and [Templates](../templates/) for how they're wired up.

## Project file

`project.latte` is a Groovy DSL that the Latte CLI reads. The template wires up:

- A compile dependency on `org.lattejava:web` (which transitively pulls in `org.lattejava:http` and `org.lattejava:jwt`)
- A test dependency on TestNG
- The `dependency`, `idea`, `java`, `java-testng`, and `release-git` plugins
- Targets for `clean`, `build`, `test`, `run`, `int`, and `release`

The most relevant target for development is `run`, which builds the project and launches the `Main` class:

~~~~ groovy
target(name: "run", description: "Runs the web server", dependsOn: ["build"]) {
  java.run(main: "org.example.hello_web.Main")
}
~~~~

Because the `java` plugin builds the project as a module (a `module-info.java` is present), your application runs on the module path with `org.lattejava.web` and its transitive dependencies resolved as named modules. That's what makes `import module org.lattejava.web` work in `Main.java`. See the [Java Plugin](../../../cli/docs/plugin-java/) docs for the `run` target's full options.

## Common commands

| Command | What it does |
|---|---|
| `latte run` | Runs `Main.java` directly (fastest dev loop) |
| `latte build` | Compiles and JARs the project |
| `latte test` | Compiles, JARs, and runs TestNG |
| `latte clean` | Removes build artifacts |

For details on the Latte CLI, see the [CLI documentation](../../../cli/docs/).
