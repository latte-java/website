---
title: "Documentation"
layout: docs
description: "Latte database documentation — zero-dependency SQL migrations for Java"
weight: 1
---

Welcome to the Latte `database` documentation. `database` is a SQL migration runner: you write plain `.sql` files named after a version, and `Migrator` applies the ones that haven't run yet, in order, recording each one in a version table. It has no runtime dependencies beyond the JDK and your JDBC driver.

## Getting Started

- [Getting Started](getting-started/) — add the dependency, write your first migration, and run it
- [Migration Files](migration-files/) — naming, ordering, checksums, and where the scripts can live

## Running Migrations

- [Running Migrations](running-migrations/) — on application startup, from the Latte CLI, and the version table itself
- [SQL Scripts](sql-scripts/) — how a script is split into statements, and the dialect-specific syntax that's supported

## Guarantees

- [Transactions & Concurrency](transactions-and-concurrency/) — what's atomic, and how concurrent JVMs are serialized
- [Failures](failures/) — every way a run can fail, what state it leaves behind, and how to recover
