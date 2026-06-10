---
layout: docs
title: Catch-All Members
description: Capturing unknown JSON keys with @JSONCatchAll.
weight: 70
---

`@JSONCatchAll` marks one member as the bucket for any JSON keys that don't map to a declared member. Instead of dropping unknown keys (lenient mode) or throwing on them ([`strict` mode](../field-customization/#output-options)), the parser collects them into a map — and writes them back out on serialize.

The annotated member must be typed exactly `Map<String, Object>`, and a type may have at most one catch-all.

```java
@JSON
public record Response(String id, int code, @JSONCatchAll Map<String, Object> extras) {}
```

## Round-trip

Given this input:

```json
{"id":"a","code":7,"note":"hi","tags":[1,2],"meta":{"k":"v"}}
```

`id` and `code` populate their components; everything else lands in `extras`:

- `extras.get("note")` → `"hi"`
- `extras.get("tags")` → `[1, 2]` (an `ArrayList`)
- `extras.get("meta")` → `{"k":"v"}` (a `LinkedHashMap`)

On serialize, the catch-all entries are spread back out as top-level keys after the declared members, so the value round-trips. The map is a `LinkedHashMap`, so insertion order is preserved.

Captured values take their natural JSON shapes — `String`, `Long` (or `BigInteger` for very large integers), `BigDecimal`, `Boolean`, `null`, `ArrayList<Object>` for arrays, and `LinkedHashMap<String, Object>` for nested objects. Nested `@JSON` types are not reconstructed inside a catch-all; an object becomes a plain map.

## Interaction with `strict`

A catch-all captures unknown keys **regardless of `strict`** — there are no "unknown" keys left to reject once everything has a home. Use `@JSON(strict = true)` without a catch-all when you want unknown keys to be an error; use a catch-all when you want to preserve them.

## Null handling

A `"key": null` on input is captured as a `null` entry in the map. Whether it's written back out follows the type's [`omitNulls`](../field-customization/#output-options) setting — set `@JSON(omitNulls = false)` to round-trip explicit nulls.

## Validation

- The member's type must be exactly `Map<String, Object>`.
- At most one `@JSONCatchAll` per type.
- `@JSONCatchAll` cannot be combined with `@JSONField` on the same member (a catch-all has no single key to name).
