---
title: "Documentation"
layout: docs
description: "Latte JSON documentation — compile-time JSON serialization for Java 25"
weight: 1
---

Welcome to the Latte `json` documentation. `json` is a compile-time JSON library: you annotate a record, class, or sealed interface with `@JSON`, and an annotation processor generates a companion serializer/deserializer for it during compilation. There's no reflection, and nothing from the library ends up on your runtime path.

## Getting Started

- [Getting Started](getting-started/) — wire up the processor, annotate your first type, and round-trip it
- [Supported Types](supported-types/) — the scalar type mapping, collections, enums, and nested objects

## Defining Types

- [Records, Classes & Beans](records-classes-and-beans/) — the three kinds of `@JSON` types and how each is read and written
- [Field Customization](field-customization/) — `@JSONField` and the `@JSON` output options (`omitNulls`, `strict`)
- [Naming Strategies](naming-strategies/) — converting Java names to JSON keys
- [Dates & Times](dates-and-times/) — ISO strings, custom patterns, and epoch integers

## Advanced

- [Catch-All Members](catch-all/) — capturing unknown JSON keys with `@JSONCatchAll`
- [Polymorphism](polymorphism/) — discriminated sealed hierarchies with `@JSONTypeInfo` and `@JSONSubtype`
- [Performance](performance/) — JMH benchmarks against Jackson and Gson
