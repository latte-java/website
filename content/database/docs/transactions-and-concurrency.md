---
layout: docs
title: Transactions & Concurrency
description: What a run does atomically, and how simultaneous runs are serialized.
weight: 50
---

## What a run does

In order, every call to `migrate()`:

1. Acquires the migration lock — before anything else, including before the version table exists.
2. Creates the version table if it isn't there.
3. Reads the recorded versions and verifies their checksums against the files.
4. Applies each pending migration, in SemVer order, in its own transaction.
5. Releases the lock and restores the connection's original `autoCommit` setting.

Step 5 runs on every exit path, including failures. If releasing the lock or restoring the connection fails after a migration already failed, that secondary problem is attached to the original exception as a suppressed exception rather than replacing it — the error you see is always the one that actually broke.

## One transaction per migration

Each migration runs inside its own transaction, and the row recording it in the version table is written and committed as part of that same transaction. A migration is therefore either fully applied *and* recorded, or neither.

Migrations are **not** wrapped in one big transaction together. If the third of five migrations fails, the first two stay applied and recorded, and the run stops — the fourth and fifth are never attempted. Fixing the third migration and running again picks up exactly where it left off.

### PostgreSQL

PostgreSQL has transactional DDL, so a failed migration rolls back completely. The schema is exactly as it was before that migration started, and nothing is recorded. This is the clean case: fix the script and run again.

### MySQL and MariaDB

MySQL commits implicitly on DDL. A migration that creates two tables and fails on the second leaves the first table behind — the transaction can't take it back. The version row is still not written, so the migration is considered un-applied and will run again from the top once you fix it.

That has a practical consequence: on MySQL, a migration script should be written so that re-running it after a partial failure works. Prefer `CREATE TABLE IF NOT EXISTS` and similar guarded forms, and keep the number of DDL statements per migration small — a single statement per file is the extreme version of this, and it's not a bad default for MySQL. Otherwise, expect to clean up the partial schema by hand before retrying.

### Other databases

H2, HSQLDB, and anything else the runner doesn't recognize behave according to their own DDL transaction semantics. The runner treats them the same way structurally — one transaction per migration, with the version row committed alongside.

## Concurrent runs

Every instance of an application calling `migrate()` on startup means simultaneous runs are normal, not exceptional. The runner serializes them with a lock held in the database for the entire run.

**PostgreSQL** takes a session advisory lock — `pg_advisory_lock` on a 64-bit key derived from the version table's name. **MySQL and MariaDB** take a named lock — `GET_LOCK('<table>.migration', -1)`. Both wait indefinitely rather than failing: a second JVM arriving mid-run blocks until the first finishes, then proceeds and finds nothing left to do. Both locks are session-scoped, so if a JVM dies mid-migration the database releases the lock as soon as the connection drops — there's no stale lock row to clean up by hand.

Because the lock key is derived from the version table name, two applications that share a database but use different version tables never block each other. Two that use the same table are serialized, which is what you want if they're managing the same schema.

**H2, HSQLDB, and unrecognized databases** have no session-scoped lock available, so protection there is best-effort: each migration re-checks the version table under `SELECT ... FOR UPDATE` inside its own transaction and skips the migration if another JVM has already recorded it, with the primary key on `version` as the hard backstop against a double-apply. That's adequate for the single-process development and test use these databases usually see; don't rely on it for a cluster.

### Startup implications

The lock is held for the whole run, not per migration. If one instance is applying a migration that takes four minutes, every other instance's `migrate()` call blocks for those four minutes, and if you migrate during startup, that's four minutes of blocked startup. Long data backfills are usually better run as their own step rather than as a startup migration.

### One connection, one run

A `Migrator` uses the `Connection` you give it exclusively for the duration of the run — it changes `autoCommit`, commits, and rolls back on it. Don't share that connection with other work while `migrate()` is executing, and don't call `migrate()` from two threads on the same connection. Separate connections (from a pool, or from different JVMs) are exactly what the lock is there to handle.
