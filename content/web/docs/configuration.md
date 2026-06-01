---
layout: docs
title: Configuration
description: Resolving named settings from environment variables, system properties, and properties files with typed getters.
weight: 15
---

## Overview

`Configuration` resolves named settings from a layered lookup chain. For each setting name, the sources are consulted in order and the **first defined value wins**:

1. The environment variable whose name matches the setting name **verbatim**.
2. The environment variable whose name is the setting name **uppercased** with every non-alphanumeric character replaced by an underscore (so `my-app.some-setting` becomes `MY_APP_SOME_SETTING`).
3. The Java **system property** with the same name as the setting.
4. Each **properties file** passed to the constructor that exists on disk, consulted in the order they were supplied.

Because environment variables and system properties are consulted before files, they override values defined in your properties files — which is what makes per-deployment overrides work without touching the file on disk.

If a setting is not defined in any source, the no-default getters return `null` and the default-value getters return the supplied default.

## Constructing

There are four constructors:

| Constructor | Sources | Required settings |
| --- | --- | --- |
| `Configuration()` | env vars + system properties only | none |
| `Configuration(Path...)` | env vars + system properties + files | none |
| `Configuration(List<String> requiredSettings)` | env vars + system properties only | validated |
| `Configuration(List<String> requiredSettings, Path...)` | env vars + system properties + files | validated |

### Properties files

Files are read with `Properties.load(InputStream)` and consulted in the order they are supplied; the first file to define a setting wins. Paths that **do not exist are silently ignored**, so you can list optional override files freely. A file that exists but **cannot be read** throws an `UncheckedIOException`.

### Required settings

When you pass a list of required settings, they are validated **at construction time**. If any required setting is not defined in any source, the constructor throws an `IllegalStateException` naming the missing settings:

~~~~ text
Missing required configuration settings [db.url, oidc.client-secret]
~~~~

This lets your application fail fast at startup rather than discovering a missing secret deep in a request.

## A realistic example

~~~~ java
import module java.base;
import module org.lattejava.web;

void main() {
  var config = new Configuration(
      List.of("db.url", "oidc.client-secret"),
      Paths.get("config/app.properties"),
      Paths.get("config/local.properties"));

  String dbUrl = config.get("db.url");
  int port = config.getInteger("server.port", 8080);
  boolean debug = config.getBoolean("debug", false);

  new Web().start(port);
}
~~~~

Here `config/app.properties` holds the committed defaults and `config/local.properties` (which may not exist) holds machine-specific overrides that take precedence. Both `db.url` and `oidc.client-secret` are required, so the constructor throws if neither files, env vars, nor system properties define them.

## Environment variable override naming

The second lookup step normalizes the setting name before checking the environment. Every non-alphanumeric character becomes `_` and letters are uppercased:

| Setting name | Normalized env var |
| --- | --- |
| `server.port` | `SERVER_PORT` |
| `my-app.some-setting` | `MY_APP_SOME_SETTING` |
| `oidc.client-secret` | `OIDC_CLIENT_SECRET` |

So a setting read as `config.get("oidc.client-secret")` can be overridden in any environment with:

~~~~ text
export OIDC_CLIENT_SECRET=super-secret-value
~~~~

without changing the properties file. Note that the verbatim env var (step 1) is checked first, so an env var literally named `oidc.client-secret` would also match if your shell allows it.

## Typed getters

Each getter has a no-default variant (returns `null` / boxed null when undefined) and a default variant. The default variants of `getBoolean` and `getInteger` return primitives.

| Method | Returns | When undefined |
| --- | --- | --- |
| `get(String name)` | `String` | `null` |
| `get(String name, String defaultValue)` | `String` | `defaultValue` |
| `getInteger(String name)` | `Integer` | `null` |
| `getInteger(String name, int defaultValue)` | `int` | `defaultValue` |
| `getBoolean(String name)` | `Boolean` | `null` |
| `getBoolean(String name, boolean defaultValue)` | `boolean` | `defaultValue` |
| `getBigDecimal(String name)` | `BigDecimal` | `null` |
| `getBigDecimal(String name, BigDecimal defaultValue)` | `BigDecimal` | `defaultValue` |
| `getBigInteger(String name)` | `BigInteger` | `null` |
| `getBigInteger(String name, BigInteger defaultValue)` | `BigInteger` | `defaultValue` |

A few behaviors worth knowing:

- **Booleans** parse via `Boolean.valueOf` / `Boolean.parseBoolean`: a value is `true` only when it equals `"true"` ignoring case. Everything else is `false`.
- **Numeric getters** throw `NumberFormatException` if the setting is defined but the value cannot be parsed as the requested type. An undefined setting never throws — it returns `null` or the default.

~~~~ java
String region = config.get("aws.region", "us-east-1");
int poolSize = config.getInteger("db.pool-size", 10);
boolean cacheEnabled = config.getBoolean("cache.enabled", true);
BigDecimal rate = config.getBigDecimal("billing.rate");          // null if unset
BigInteger maxId = config.getBigInteger("import.max-id");        // null if unset
~~~~

## Related

- [Getting Started](../getting-started/) — wiring up a `Web` application
- [OIDC Authentication](../oidc/) — a common consumer of configured client IDs and secrets
