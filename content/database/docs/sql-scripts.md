---
layout: docs
title: SQL Scripts
description: How a migration script is split into statements, and the dialect syntax that's understood.
weight: 40
---

## One file, many statements

A migration script is executed as a sequence of statements, split on `;`. Each statement is sent to the database on its own, in order, over plain JDBC. Statements can be anything the database accepts — DDL, DML, `GRANT`, whatever — and no parameter binding is involved: the SQL in the file is the SQL that runs.

```sql
CREATE TABLE users (
  id   BIGSERIAL PRIMARY KEY,
  name TEXT NOT NULL
);

CREATE INDEX users_name ON users (name);

INSERT INTO users (name) VALUES ('root');
```

The final statement doesn't need a trailing semicolon. Whitespace and comments between statements are not sent as statements of their own, so a file that ends with a comment or a stray semicolon is fine.

Because scripts run over JDBC rather than through `psql` or the `mysql` client, client meta-commands are not available. psql's backslash commands (`\i`, `\copy`, `\c`) and `mysql`'s `source` aren't SQL and won't work here. The one exception is MySQL's `DELIMITER`, which is described below.

## Splitting is syntax-aware

The splitter is a real lexer, not a search for semicolons. A `;` inside a string, an identifier, or a comment does not end a statement:

```sql
-- This comment; contains a semicolon and doesn't split anything
INSERT INTO messages (body) VALUES ('hello; world');

/* Block comments
   spanning lines; are handled too */
CREATE TABLE "weird;name" (id INT);
```

Comments are passed through to the database as part of the statement they precede, so anything you write in the script is visible in the database's own logs and error messages.

## Dialects

The dialect is detected from the JDBC driver's reported product name: PostgreSQL, MySQL/MariaDB, or a generic ANSI-ish mode for everything else (H2, HSQLDB, and any database not recognized). The dialect only changes what the splitter understands — it never rewrites your SQL.

| Syntax                                     | PostgreSQL | MySQL / MariaDB | Other |
|--------------------------------------------|------------|------------------|-------|
| `-- line comments`                          | Yes        | Yes              | Yes   |
| `# line comments`                           | No         | Yes              | No    |
| `/* block comments */`                      | Yes        | Yes              | Yes   |
| Nested `/* /* block comments */ */`         | Yes        | No               | No    |
| `'strings'` with `''` doubling              | Yes        | Yes              | Yes   |
| `"quoted identifiers"` with `""` doubling   | Yes        | Yes              | Yes   |
| `` `backtick identifiers` ``                | No         | Yes              | No    |
| Backslash escapes inside strings            | In `E''` only | Yes           | No    |
| `$$` / `$tag$` dollar quoting               | Yes        | No               | No    |
| `DELIMITER` directive                       | No         | Yes              | No    |

## PostgreSQL function bodies

Dollar-quoted strings are understood, including tagged ones, so a function body full of semicolons stays a single statement:

```sql
CREATE FUNCTION touch_updated_at() RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

Tags work the same way, and nesting a `$$` body inside a `$outer$` block is fine:

```sql
DO $migrate$
BEGIN
  IF NOT EXISTS (SELECT 1 FROM pg_type WHERE typname = 'role') THEN
    CREATE TYPE role AS ENUM ('OWNER', 'CONTRIBUTOR');
  END IF;
END;
$migrate$;
```

`E''` strings are recognized too, so `E'it\'s'` doesn't end the string early.

## MySQL stored programs

MySQL has no dollar quoting, so a routine body with semicolons in it needs the client-side `DELIMITER` directive — the same one you'd use in the `mysql` client:

```sql
DELIMITER //

CREATE PROCEDURE add_user(IN user_name VARCHAR(255))
BEGIN
  INSERT INTO users (name) VALUES (user_name);
  UPDATE stats SET user_count = user_count + 1;
END //

DELIMITER ;

INSERT INTO users (name) VALUES ('root');
```

`DELIMITER` is handled entirely by the splitter and is never sent to the server — it just changes which token ends a statement from that point on. It has to appear at the start of a line and between statements, and `DELIMITER ;` puts things back to normal.

## Statement errors

When a statement fails, the error names the migration, the statement's position in the file, and the line it started on, with the database's own error attached as the cause:

```
Migration [1.2.0] failed at statement [3] (line 27): ALTER TABLE users ADD COLUMN ...
```

Statements are numbered from zero, in the order they appear in the file. What happens to the statements that already succeeded depends on the database — see [Transactions & Concurrency](../transactions-and-concurrency/).
