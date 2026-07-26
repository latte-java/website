---
layout: docs
title: Failures
description: Every way a migration run can fail, what it leaves behind, and how to recover.
weight: 60
---

## Exceptions

`migrate()` throws `MigrationException`, a checked exception. `ChecksumException` is its one subclass, thrown when the files no longer match what was recorded. Catching `MigrationException` catches everything.

When a failure happens while cleaning up — releasing the lock, restoring the connection — that secondary failure is attached to the original exception with `addSuppressed`, so the exception you catch is always the one that describes what actually went wrong. Failures coming from the database carry the driver's `SQLException` as their cause.

## Failures that apply nothing

These are all detected before any migration runs, so the database is untouched when they're thrown.

### A migration file is misnamed

```
Migration file [1.0.sql] is not named <semver>.sql: Invalid SemVer version [1.0]
```

Rename the file. See [Migration Files](../migration-files/#naming) for what counts as a valid name — the most common surprises are two-component versions, a `v` prefix, and build metadata.

### Two migrations share a version

```
Two migrations have the same version [1.0.0]
```

Two files parsed to the same version. On the classpath this can mean a stale copy of the scripts in a second JAR or an old exploded output directory, not two files in your source tree.

### The migrations location doesn't exist

```
Migration directory [/path/to/db] does not exist
Classpath location [db] does not exist
```

An empty location is fine — a location that isn't there at all is not. On the classpath, this usually means the resource directory wasn't packaged into the artifact.

### A migration is out of order

```
Out-of-order migrations [1.0.5] have versions lower than the highest applied version [1.1.0]
```

Some pending migration sits below the high-water mark of what's already been applied — almost always two branches that each added a migration and merged. Rename the straggler to a version above the highest applied one. Don't renumber the migration that already ran: it's recorded under its current name.

### A checksum doesn't match

```
Migration checksum verification failed:
[1.0.0] checksum mismatch - recorded [3f2b…] but the file is now [9c81…]
[1.1.0] was applied but its migration file no longer exists
```

Every mismatch is listed, not just the first. This means one of two things:

- **The file changed.** Someone edited or reformatted a migration that had already run. The database was migrated with the old contents, so the file no longer describes the database. Restore the original bytes — `git checkout` on the file is usually all it takes — and put the change in a new migration with a higher version.
- **The file is gone.** A migration that ran was deleted or didn't get packaged. Restore it. A recorded migration must keep existing, because its checksum is what proves the databases you're deploying to all share the same history.

If you're certain the edit was cosmetic and every database that has run this migration is genuinely in the state the new file describes, you can re-point the record at the new contents by updating the checksum by hand — everywhere it's been applied:

```bash
shasum -a 256 src/main/resources/db/1.0.0.sql
```

```sql
UPDATE versions SET checksum = '<the hash>' WHERE version = '1.0.0';
```

This is an escape hatch, and it's exactly the drift the checksums exist to prevent. Reach for it rarely and never as a matter of routine.

## Failures during a migration

### A statement fails

```
Migration [1.2.0] failed at statement [3] (line 27): ALTER TABLE users ADD COLUMN ...
```

The message names the migration, the zero-based index of the statement in the file, and the line the statement started on; the database's own error is the cause. Migrations before this one stay applied and recorded; migrations after it are never attempted; this one is not recorded.

What's left behind depends on the database. On PostgreSQL, nothing — the transaction rolled back the DDL along with everything else. On MySQL, DDL that already committed implicitly stays. See [Transactions & Concurrency](../transactions-and-concurrency/#one-transaction-per-migration) for how to write migrations that survive that.

Fix the script and run again. Because the failed migration was never recorded, its checksum was never recorded either — editing it is not only allowed here, it's the fix.

### The version table has a bad row

```
The [versions] table contains a row with an invalid version [1.0]
```

Something wrote a row that isn't a valid SemVer version — nearly always a hand-written `INSERT`. Correct or remove the row.

### The lock can't be acquired

```
Unable to acquire the MySQL migration lock [versions.migration]
```

The lock calls wait indefinitely rather than timing out, so this is rare — it means the database refused the lock outright rather than that someone else holds it. A run waiting on another JVM's migration doesn't fail; it blocks.

## Adopting an existing database

To start using migrations on a database that already has a schema, write the migration that describes what's already there — then tell the runner it's already been applied, so it doesn't try to create the tables a second time.

Create the table and insert the record yourself:

```sql
CREATE TABLE versions (
    version           VARCHAR(255) NOT NULL PRIMARY KEY,
    checksum          CHAR(64)     NOT NULL,
    installed_instant BIGINT       NOT NULL
);
```

```bash
shasum -a 256 src/main/resources/db/1.0.0.sql
```

```sql
INSERT INTO versions (version, checksum, installed_instant)
VALUES ('1.0.0', '<the hash>', 1753400000000);
```

The checksum must be the lowercase hex SHA-256 of the file's raw bytes and `installed_instant` is epoch milliseconds — any plausible value works there, it's recorded for humans, not used in any decision.

Do this on every existing database. From then on, new environments get the same schema by running `1.0.0.sql` for real, and everybody moves forward together on `1.1.0.sql` and up.
