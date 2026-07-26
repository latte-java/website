---
layout: docs
title: Running Migrations
description: Applying migrations on application startup or from your build, and the version table that records them.
weight: 30
---

## The API

There is exactly one entry point:

```java
List<Version> applied = new Migrator(connection, "db").migrate();
```

`migrate()` verifies the checksums of everything already applied, then applies each pending migration in order and returns the versions it applied — in the order they were applied, empty when the database is already up to date. It throws the checked `MigrationException` when anything goes wrong, so failures can't be ignored by accident.

There's no dry-run mode and no separate "info" or "validate" call. If you want to know what a run would do, run it: it's idempotent, and the return value tells you what happened.

### The connection is yours

`Migrator` never closes the `Connection` you give it, and it restores the connection's original `autoCommit` setting before it returns or throws. A connection borrowed from a pool comes back the way you lent it, so the normal shape is a try-with-resources around the call:

```java
try (Connection connection = dataSource.getConnection()) {
  new Migrator(connection, "db").migrate();
}
```

## On application startup

The migration runner is designed to be called while your application boots, before anything else touches the schema. Because concurrent runs are [serialized with a database lock](../transactions-and-concurrency/), it's safe for every instance of a horizontally scaled application to do this — one instance applies the migrations, the others wait and then find nothing to do.

Here's the pattern in a service that owns the connection pool:

```java
import module java.base;
import module org.lattejava.database;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

public class DatabaseService {
  private static final System.Logger LOG = System.getLogger(DatabaseService.class.getName());
  private final HikariDataSource dataSource;

  public DatabaseService(Configuration config) {
    HikariConfig hikariConfig = new HikariConfig();
    hikariConfig.setJdbcUrl(config.get("db.url"));
    hikariConfig.setUsername(config.get("db.username"));
    hikariConfig.setPassword(config.get("db.password"));
    this.dataSource = new HikariDataSource(hikariConfig);

    // Apply any pending migrations (classpath db/*.sql) before anything else touches the schema
    try (Connection connection = dataSource.getConnection()) {
      List<Version> applied = new Migrator(connection, "db").migrate();
      if (!applied.isEmpty()) {
        LOG.log(System.Logger.Level.INFO, "Applied database migrations " + applied);
      }
    } catch (MigrationException | SQLException e) {
      throw new IllegalStateException("Unable to migrate the database", e);
    }

    // ... build the DSLContext, repositories, and so on
  }
}
```

Two things worth copying from that example:

- **Migrate before you expose anything.** Do it in the constructor of whatever owns the data layer, or as the first thing in `main`, so the rest of the application can't observe a half-migrated schema.
- **Fail the startup.** A database that couldn't be migrated is not a database your application should serve traffic against. Wrapping the failure in an unchecked exception and letting the process die is usually correct — a container will restart it, and a genuinely broken migration will keep failing loudly instead of running against the wrong schema.

The library writes nothing to a log itself and has no logging dependency. It returns the applied versions and throws on failure; what gets logged is entirely up to you.

### Credentials

Migrations run over whatever connection you hand them, so they use the credentials of that connection. Applying DDL usually needs more privileges than the application's steady-state queries do. If you'd rather your runtime user not hold `CREATE TABLE`, migrate with a separate `DataSource` (or a plain `DriverManager.getConnection` call) that uses a migration user, and close it once `migrate()` returns.

## From the build

The Latte CLI's [database plugin](/cli/docs/plugin-database/#running-migrations) runs the same migrations from a `project.latte` target, reading them from a directory rather than the classpath:

```groovy
target(name: "migrate", description: "Applies the pending database migrations") {
  database.settings.type = "postgresql"
  database.settings.name = "myapp"
  database.migrate()
}
```

The plugin defaults to `src/main/resources/db` and the `versions` table — the same location and table this library defaults to — so a project can migrate from the build during development and from the classpath in production without the two disagreeing about what's been applied.

## The version table

Applied migrations are recorded in a table the runner creates the first time it runs:

```sql
CREATE TABLE versions (
    version           VARCHAR(255) NOT NULL PRIMARY KEY,
    checksum          CHAR(64)     NOT NULL,
    installed_instant BIGINT       NOT NULL
)
```

- `version` — the migration's version, exactly as it appeared in the file name
- `checksum` — lowercase hex SHA-256 of the script's raw bytes
- `installed_instant` — when it was applied, as epoch milliseconds UTC

The primary key on `version` is what makes a double-apply impossible even in the rare case where two JVMs get past the lock at the same moment.

### Using a different table

Both constructors take the table name as a final argument:

```java
new Migrator(connection, "db", "schema_versions").migrate();
new Migrator(connection, Path.of("db/migrations"), "schema_versions").migrate();
```

The migration lock is keyed on the table name, so two components that share a database but keep separate version tables migrate independently and don't block each other. That's the way to let two applications own two sets of tables in one database — give each its own scripts and its own version table.

Nothing else in the library reads or writes the table, so you can query it directly to see what's been applied:

```sql
SELECT version, installed_instant FROM versions ORDER BY installed_instant;
```

Editing it by hand is a way to lie to the runner about what's been applied. It's occasionally the right escape hatch — see [Failures](../failures/) — but it should be a deliberate act, not a routine one.
