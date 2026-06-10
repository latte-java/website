---
layout: docs
title: Naming Strategies
description: Converting Java member names to JSON keys.
weight: 50
---

The `naming` element on `@JSON` controls how each member name is converted to its JSON wire key:

```java
@JSON(naming = NamingStrategy.SNAKE_CASE)
public record Account(String userName, String httpStatusCode) {}
// → {"user_name":"...","http_status_code":"..."}
```

The converter splits the Java identifier into words — inserting a boundary before each capitalized word and at the tail of an acronym run (`userID` → `user` + `ID`), with digits attaching to the preceding word — then lowercases and rejoins per the strategy.

| Strategy             | `firstName`  | `userID`    | `httpStatusCode`    |
|----------------------|--------------|-------------|---------------------|
| `IDENTITY` (default) | `firstName`  | `userID`    | `httpStatusCode`    |
| `CAMEL_CASE`         | `firstName`  | `userId`    | `httpStatusCode`    |
| `PASCAL_CASE`        | `FirstName`  | `UserId`    | `HttpStatusCode`    |
| `SNAKE_CASE`         | `first_name` | `user_id`   | `http_status_code`  |
| `KEBAB_CASE`         | `first-name` | `user-id`   | `http-status-code`  |

`IDENTITY` leaves the Java name untouched. The others normalize acronyms, so `userID` becomes `userId` / `user_id` rather than preserving the original casing.

## Per-member overrides

A [`@JSONField(name = "...")`](../field-customization/) on an individual member always wins over the type's strategy for that member:

```java
@JSON(naming = NamingStrategy.SNAKE_CASE)
public record Event(
    String userName,                          // → "user_name"
    @JSONField(name = "X-Request-ID") String requestId) {}  // → "X-Request-ID"
```

The resulting wire keys are validated at compile time: if the strategy and overrides produce two members with the same key, the build fails rather than silently dropping one.
