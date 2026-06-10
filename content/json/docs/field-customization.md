---
layout: docs
title: Field Customization
description: Per-member control with @JSONField, and the @JSON output options.
weight: 40
---

## `@JSONField`

Annotate any record component, `@JSONConstructor` parameter, or JavaBean accessor/field with `@JSONField` to override how that member is represented. Every element is optional; a member with no `@JSONField` uses the type's [naming strategy](../naming-strategies/) and round-trips in both directions.

| Element     | Type            | Default | Description                                                                                              |
|-------------|-----------------|---------|----------------------------------------------------------------------------------------------------------|
| `name`      | `String`        | `""`    | Explicit JSON key for this member. Overrides the type's `@JSON(naming)` strategy.                         |
| `ignore`    | `boolean`       | `false` | Drop the member from both directions — never serialized, never read on input.                            |
| `readOnly`  | `boolean`       | `false` | Serialize only. Written to JSON but never populated from input.                                          |
| `writeOnly` | `boolean`       | `false` | Deserialize only. Read from input but omitted from the JSON output.                                      |
| `format`    | `String`        | `""`    | A `java.time` pattern for a date/time member — see [Dates & Times](../dates-and-times/#custom-patterns-with-format). |
| `instant`   | `InstantFormat` | `ISO`   | Wire representation for an `Instant` — see [Dates & Times](../dates-and-times/#epoch-integers-with-instant). |

```java
@JSON(naming = NamingStrategy.SNAKE_CASE)
public record Account(
    @JSONField(name = "id") String accountId,      // → "id" (strategy ignored for this field)
    String userName,                                // → "user_name" (from the strategy)
    @JSONField(writeOnly = true) String password,   // read from input, never serialized
    @JSONField(readOnly = true) Instant createdAt,  // serialized, never read back
    @JSONField(ignore = true) String scratch) {}    // absent from JSON entirely
```

A `readOnly` (or `ignore`) member that *does* appear on input is handled like any unknown key: dropped under the default lenient mode, or rejected under `@JSON(strict = true)` (see below).

### Validation

Contradictory combinations fail the build rather than silently misbehaving:

- `readOnly = true` together with `writeOnly = true` — that's just `ignore`.
- `ignore = true` together with any other element — the others would have no effect.
- `format` or a non-default `instant` on a member of the wrong type — see [Dates & Times](../dates-and-times/).

Wire keys are also validated: a duplicate key (after the naming strategy and any `name` overrides are applied) or a key with invalid characters fails compilation.

## Output options

Two elements on the `@JSON` annotation itself control the wire form for the whole type.

| Element      | Type      | Default | Effect                                                                 |
|--------------|-----------|---------|------------------------------------------------------------------------|
| `omitNulls`  | `boolean` | `true`  | When `true`, a `null` member is omitted from the JSON object entirely; when `false`, it's written as `"key": null`. |
| `strict`     | `boolean` | `false` | When `false`, unknown JSON keys are ignored on input; when `true`, an unknown key throws `JSONProcessingException`. |

```java
@JSON(omitNulls = false, strict = true)
public record Settings(String theme, String locale) {}
```

`omitNulls` applies to object members only — elements inside an array are always written, so a `null` in a `List` stays `null`. A type that declares a [catch-all member](../catch-all/) captures unknown keys regardless of `strict`.

(The third `@JSON` element, `naming`, has its own page: [Naming Strategies](../naming-strategies/).)
