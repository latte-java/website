---
layout: docs
title: Migration Files
description: How migrations are named, ordered, checksummed, and located.
weight: 20
---

## Naming

A migration is a file named `<semver>.sql`. The part before the extension is parsed as a strict [SemVer 2.0.0](https://semver.org) version, and that version is the migration's identity — it's what gets recorded, compared, and ordered.

| File                  | Valid | Why                                       |
|-----------------------|-------|-------------------------------------------|
| `1.0.0.sql`           | Yes   |                                           |
| `0.1.0.sql`           | Yes   |                                           |
| `1.36.3.sql`          | Yes   |                                           |
| `2.0.0-beta.1.sql`    | Yes   | Pre-release identifiers are allowed       |
| `1.0.sql`             | No    | SemVer requires all three components      |
| `v1.0.0.sql`          | No    | No prefixes                               |
| `01.0.0.sql`          | No    | No leading zeros                          |
| `20260725.sql`        | No    | Not a SemVer version                      |
| `1.0.0+build.5.sql`   | No    | Build metadata is rejected — see below    |

Build metadata (the `+…` suffix) is rejected deliberately. SemVer says it is ignored when determining precedence, so `1.0.0+a.sql` and `1.0.0+b.sql` would be two distinct files that tie exactly — there'd be no defined order between them and no way to tell which one a recorded version referred to.

Anything in the migrations location that doesn't end in `.sql` is ignored, so a `README.md` or an editor backup file sitting next to the scripts is harmless. Subdirectories are not scanned — only the files directly at the location are read.

Two files that parse to the same version — for example a `1.0.0.sql` in an exploded classpath directory and another in a JAR at the same location — fail the run.

## Ordering

Migrations are applied in SemVer precedence order, which is not the same as sorting the file names as text. Major, minor, and patch are compared as numbers, and a pre-release version sorts *below* the release it leads to:

```
1.0.0-alpha  <  1.0.0-alpha.1  <  1.0.0-alpha.beta  <  1.0.0-beta  <
1.0.0-beta.2  <  1.0.0-beta.11  <  1.0.0-rc.1  <  1.0.0  <  1.0.1  <  1.10.0
```

Within the pre-release part, dot-separated identifiers are compared one at a time: numeric identifiers are compared numerically (so `beta.11` is above `beta.2`, unlike a text sort) and sort below alphanumeric ones, and when one is a prefix of the other the longer one wins.

Once a version has been applied, every new migration must be *above* it. If a pending migration has a version lower than the highest version already recorded, the run fails and applies nothing — this is what catches the merge where two branches each added a migration and one of them got a number that's already been passed. Fix it by renaming the straggler to a version above the current high-water mark.

## Checksums

Every migration is hashed with SHA-256 over the file's raw bytes, and that hash is stored alongside the version when the migration is applied. Before anything runs, each recorded version is checked against the file it came from.

This means **an applied migration can never be edited**. Not to fix a typo in a comment, not to reformat it, not to change whitespace — any byte-level change produces a different hash and the next run fails with a `ChecksumException` rather than quietly leaving your databases inconsistent with each other. Deleting an applied migration fails the same way. Every mismatch is collected and reported together, so a single run tells you about all of them.

The right response is always the same: leave the applied file alone and add a new migration with a higher version.

Files are decoded as UTF-8 when they're run, but the checksum is over the raw bytes, so re-saving a file in a different encoding or with different line endings also changes it.

## Where migrations live

`Migrator` reads scripts from one of two places, chosen by which constructor you call.

### The classpath

Passing a `String` treats it as a classpath location:

```java
new Migrator(connection, "db").migrate();
```

Leading and trailing slashes are stripped, so `"db"`, `"/db"`, and `"db/"` are all the same location. The location is resolved with the thread's context class loader (falling back to the one that loaded the library), and **every** match is used — an exploded directory and a JAR that both contain `db/` contribute their scripts to the same run. Reading from inside a JAR works exactly like reading from a directory, which is what lets a packaged application migrate itself on startup.

The location has to exist somewhere on the classpath. If nothing matches, the run fails rather than silently doing nothing.

### A directory

Passing a `Path` reads from the filesystem:

```java
new Migrator(connection, Path.of("src/main/resources/db")).migrate();
```

The directory must exist. This is the form to use from tools, tests, and build scripts that run against the working tree rather than a packaged artifact — it's also what the [CLI database plugin](/cli/docs/plugin-database/#running-migrations) uses.

Putting the scripts in `src/main/resources/db` lets you have it both ways: the build reads them from the directory, and the application reads the same files from the classpath after they've been packaged into the JAR.
