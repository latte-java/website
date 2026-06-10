---
layout: docs
title: Dates & Times
description: ISO strings, custom patterns, and epoch integers for java.time types.
weight: 60
---

`@JSON` serializes the `java.time` types — `LocalDate`, `LocalDateTime`, `OffsetDateTime`, `ZonedDateTime`, and `Instant` — as ISO-8601 strings by default. Two `@JSONField` elements change that per member.

```java
@JSON
public record Times(Instant instant, LocalDate localDate, OffsetDateTime offsetDateTime) {}
// → {"instant":"2026-06-10T17:30:00Z","localDate":"2026-06-10","offsetDateTime":"2026-06-10T17:30:00+01:00"}
```

## Custom patterns with `format`

`format` takes any `DateTimeFormatter.ofPattern(...)` string and applies it to **both** serialization and parsing of that member. It's valid on all five temporal types above.

```java
@JSON
public record Event(
    @JSONField(format = "MM/dd/yyyy") LocalDate day,                 // "06/10/2026"
    @JSONField(format = "yyyy-MM-dd'T'HH:mm:ss'Z'") Instant at) {}   // "2026-06-10T17:30:00Z"
```

The pattern is validated when your code compiles — an invalid pattern (or one containing a `"` or `\`) fails the build instead of producing broken code or throwing at runtime. For an `Instant`, the formatter is anchored to UTC so the pattern can resolve a full date-time.

## Epoch integers with `instant`

An `Instant` is often carried as a numeric epoch rather than a string. The `instant` element selects that representation; it applies only to `Instant`.

| `InstantFormat` | Wire form                                              | Example                  |
|-----------------|--------------------------------------------------------|--------------------------|
| `ISO` (default) | JSON string — ISO-8601, or the `format` pattern if set | `"2026-06-10T17:30:00Z"` |
| `EPOCH_SECONDS` | JSON integer — seconds since the epoch                 | `1781112600`             |
| `EPOCH_MILLIS`  | JSON integer — milliseconds since the epoch            | `1781112600000`          |

```java
@JSON
public record Token(
    @JSONField(instant = InstantFormat.EPOCH_SECONDS) Instant issuedAt,
    @JSONField(instant = InstantFormat.EPOCH_MILLIS) Instant expiresAt) {}
```

```json
{"issuedAt":1781112600,"expiresAt":1781112600000}
```

## Validation

`format` and a non-default `instant` are mutually exclusive — a member can be a JSON string (pattern) or a JSON integer (epoch), not both — and setting both is a compile-time error. Likewise:

- a non-default `instant` on anything but an `Instant` fails the build, and
- `format` on a non-temporal type fails the build.
