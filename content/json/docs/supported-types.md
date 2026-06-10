---
layout: docs
title: Supported Types
description: The scalar type mapping, collections, enums, and nested objects.
weight: 20
---

A `@JSON` type's members can be scalars, collections, enums, `java.time` types, or other `@JSON` types. This page is the authoritative list of what's supported.

## Scalars

| Java type                                                    | JSON wire form                          |
|--------------------------------------------------------------|-----------------------------------------|
| `boolean` / `Boolean`                                        | `true` / `false`                        |
| `byte`, `short`, `int`, `long` (and boxed)                   | integer                                 |
| `float`, `double` (and boxed)                                | number                                  |
| `BigInteger`                                                 | integer (unbounded)                     |
| `BigDecimal`                                                 | number (precision preserved)            |
| `String`                                                     | string                                  |
| `UUID`                                                       | string — canonical `8-4-4-4-12` form    |
| any `enum`                                                   | string — the constant's `name()`        |
| `Instant`, `LocalDate`, `LocalDateTime`, `OffsetDateTime`, `ZonedDateTime`, `Duration`, `Period` | string — ISO-8601 (see [Dates & Times](../dates-and-times/)) |

```java
@JSON
public record Bag(BigInteger big, BigDecimal money) {}

@JSON
public record Ids(UUID id, Color color) {}   // Color is an enum
```

Boxed numeric types and reference types accept `null`; whether a `null` is written depends on [`@JSON(omitNulls)`](../field-customization/#output-options).

## Enums

An enum member is written as its constant name and parsed back with `Enum.valueOf`. An unknown constant name on input throws `JSONProcessingException`.

```java
public enum Color { RED, GREEN, BLUE }
```

`Color.GREEN` ⇄ `"GREEN"`.

## Collections

Three container types are supported, each as a single level (the element/value type must itself be a supported scalar, enum, `java.time` type, or `@JSON` type):

| Java type   | JSON     | Deserialized into | Notes                                  |
|-------------|----------|-------------------|----------------------------------------|
| `List<E>`   | array    | `ArrayList<E>`    |                                        |
| `Set<E>`    | array    | `LinkedHashSet<E>`| insertion order preserved; dedups      |
| `Map<K, V>` | object   | `LinkedHashMap<K, V>` | insertion order preserved          |

```java
@JSON
public record Lists(List<String> tags, List<Integer> nums, List<UUID> ids) {}

@JSON
public record Maps(Map<String, Integer> counts, Map<UUID, String> labels) {}
```

```json
{"tags":["a","b"],"nums":[1,2,3],"ids":["00000000-0000-0000-0000-000000000001"]}
```

### Map keys

Because JSON object keys are always strings, a `Map` key must be a type with a canonical string form: `String`, `UUID`, any `enum`, or a `java.time` type. Enum keys use `name()`; `java.time` keys use their ISO string.

```java
@JSON
public record KeyedMaps(Map<Color, Integer> byColor, Map<Instant, String> byTime) {}
```

## Nested `@JSON` types

A member may be another `@JSON` type — directly, as a collection element, or as a map value. Each nested type just needs its own `@JSON` annotation (and must live in the same module). Nesting composes to any depth, including recursively.

```java
@JSON
public record Address(String city, String zip) {}

@JSON
public record User(String name, Address address, List<Address> previous) {}
```

```json
{"name":"alice","address":{"city":"Denver","zip":"80202"},"previous":[]}
```

## Not supported in this release

These are rejected at compile time with a clear error, rather than failing at runtime:

- **Nested collections** — `List<List<E>>`, `Map<String, List<E>>`, etc.
- **Non-string-form map keys** — `Map<Integer, V>`, `Map<SomeClass, V>`, etc.
- **Raw or wildcard type arguments** — `List` (no type argument), `List<?>`, `Map<String, ? extends Number>`.
