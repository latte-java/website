---
layout: docs
title: Getting Started
description: Add the dependency, write a migration, and apply it.
weight: 10
---

## Prerequisites

You need Java 25 and the Latte CLI. If you don't already have them:

```bash
curl -fsSL https://lattejava.org/javaenv/install | bash
```

```bash
javaenv install 25
```

```bash
javaenv global 25
```

```bash
curl -fsSL https://lattejava.org/cli/install | bash
```

You also need a JDBC driver for your database. The library itself has no dependencies — it talks to whatever `java.sql.Connection` you hand it.

## Add the dependency

Add `database` to the `compile` group in your `project.latte`, along with the JDBC driver for your database:

```groovy
dependencies {
  group(name: "compile") {
    dependency(id: "org.lattejava:database:0.1.0")
    dependency(id: "org.postgresql:postgresql:42.7.10")
  }
}
```

Then require the module:

```java
module com.example.app {
  requires org.lattejava.database;
}
```

`org.lattejava.database` requires `java.sql` transitively, so requiring it is enough to use `Connection` and friends in your own module.

## Write a migration

Migrations are ordinary SQL files named `<semver>.sql`. Create the first one in `src/main/resources/db`:

```
src/main/resources/db/
└── 1.0.0.sql
```

```sql
CREATE TABLE users (
  id    BIGSERIAL PRIMARY KEY,
  name  TEXT NOT NULL,
  email TEXT NOT NULL UNIQUE
);
```

There's no header to write and no metadata to declare. The file name is the version, and the contents are the migration.

`src/main/resources/db` is a convention rather than a requirement, but it's a useful one: it's the default the [CLI database plugin](/cli/docs/plugin-database/#migrations-directory) looks in, and because it's a resource directory the same scripts are packaged into your JAR, where the application can apply them from the classpath at startup.

## Apply it

`Migrator` takes a `Connection` and the location of the scripts:

```java
import module java.base;
import module org.lattejava.database;

try (Connection connection = dataSource.getConnection()) {
  List<Version> applied = new Migrator(connection, "db").migrate();
  System.out.println("Applied " + applied);
}
```

The first run creates the `versions` table, applies `1.0.0.sql`, and records it. Every run after that finds nothing to do and returns an empty list. Meaning this is safe to call unconditionally, on every start.

## Add the next migration

Never edit a migration that has already been applied. The runner checksums each script and will refuse to continue if one changed underneath it. Add a new file with a higher version instead:

```
src/main/resources/db/
├── 1.0.0.sql
└── 1.1.0.sql
```

```sql
ALTER TABLE users ADD COLUMN verified BOOLEAN NOT NULL DEFAULT FALSE;
```

The next run applies only `1.1.0.sql`. See [Migration Files](../migration-files/) for the full rules on naming, ordering, and checksums.

## Migrating from a directory

Passing a `String` reads the migrations from the classpath. Passing a `Path` reads them from a directory on disk instead, which is what you want for tools and tests that run against the working tree:

```java
List<Version> applied = new Migrator(connection, Path.of("src/main/resources/db")).migrate();
```

Both constructors also take the version table name as a third argument if you don't want the default `versions` table:

```java
new Migrator(connection, "db", "schema_versions").migrate();
```

## Where to go next

- [Migration Files](../migration-files/) — naming, ordering, checksums, and classpath vs. directory
- [Running Migrations](../running-migrations/) — wiring migrations into application startup or your build
- [Failures](../failures/) — what happens when a script is bad, edited, or out of order
