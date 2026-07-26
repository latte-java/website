---
title: "database"
description: "SQL migrations for Java — a zero-dependency runner that applies versioned scripts in order, at build time or on startup"
layout: landing
github: "https://github.com/latte-java/database"
---

Database utilities for Java. In the Latte world, database migration are simple. This library has no dependencies and runs real SQL scripts directly. No XML, no bespoke languages, just deliciously plain SQL.

Migrations files named using SemVer. These versions might map to the version of your application, or they might just be arbitrary version numbers (`1.0.0.sql`, `1.1.0.sql`, `2.0.0-beta.1.sql`). `Migrator` applies them in SemVer order and records each one (with its SHA-256 checksum and the time it ran) in a version table. The same call is safe to make on every start of your application, even across multiple nodes.

## Install

Add to your Latte project:

```groovy
dependency(id: "org.lattejava:database:0.1.0")
```

Or install with the CLI:

```bash
latte install org.lattejava:database:0.1.0
```

Then require the module:

```java
module com.example.app {
  requires org.lattejava.database;
}
```

## Quick Start

Put your scripts in `src/main/resources/db`:

```
src/main/resources/db/
├── 1.0.0.sql
└── 1.1.0.sql
```

Then migrate at startup, before anything else touches the schema:

```java
import module java.base;
import module org.lattejava.database;

try (Connection connection = dataSource.getConnection()) {
  List<Version> applied = new Migrator(connection, "db").migrate();
}
```

That's it! `migrate()` verifies that the migrations already applied still match what's on disk, applies everything pending in order, and returns the versions it applied or an empty list when the database is already up to date.

## Features

- **Zero dependencies** — the JDK and your JDBC driver, nothing else
- **Plain SQL** — no XML, no YAML, no DSL to learn; the scripts are the schema
- **SemVer ordering** — files are named `<semver>.sql` and applied in strict SemVer 2.0.0 order
- **Tamper detection** — every applied script is checksummed; editing one fails the next run instead of silently drifting
- **Directory or classpath** — read scripts from a directory, an exploded classpath, or a JAR
- **Safe to run everywhere** — concurrent JVMs are serialized with a database lock, so every instance can migrate on startup
- **Real SQL parsing** — dollar-quoting, `E''` strings, backticks, `DELIMITER` blocks and nested comments are all handled
- **Build-time too** — the CLI's [database plugin](/cli/docs/plugin-database/#running-migrations) runs the same migrations from your `project.latte`

[Read the documentation →](docs/)

## License

MIT
