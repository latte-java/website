---
layout: docs
title: Getting Started
description: Wire up the processor, annotate a type, and round-trip it to JSON.
weight: 10
---

## Prerequisites

You need Java 25 and the Latte CLI. If you don't already have them:

```bash
curl -fsSL https://lattejava.org/javaenv/install | bash
```

```bash
javaenv install 25
```

```bash
javaenv global 25
```

```bash
curl -fsSL https://lattejava.org/cli/install | bash
```

## Wire up the processor

`json` is an annotation processor. Latte's Java plugin discovers processors through the [`compile-processors`](/cli/docs/plugin-java/#annotation-processing) dependency group, so the entire integration is one entry in your `project.latte`:

```groovy
dependencies {
  group(name: "compile-processors") {
    dependency(id: "org.lattejava:json:0.4.0")
  }
}
```

Latte hands that artifact to `javac` via `--processor-module-path`, where the processor is auto-discovered through `ServiceLoader` and its `@JSON` annotations resolve while your sources compile. The processor runs during both main and test compilation — there's no `javac` flag to set, no `module-info` `provides` clause to write, and no separate codegen target to run.

You don't add it to the `compile` group, and nothing ships at runtime. The `@JSON` annotations are `SOURCE`-retention, and the processor emits its small set of runtime helpers directly into your module's `<your.module>.internal` package alongside the generated companions — so `org.lattejava:json` never lands on your compile classpath or your runtime path.

## Annotate a type

Annotate any record with `@JSON`:

```java
package com.example;

import module org.lattejava.json;

@JSON
public record User(String name, int age, String email) {}
```

When you compile, the processor generates a companion class named `<Type>JSON` for every `@JSON` type. Records work out of the box; classes and JavaBeans are also supported — see [Records, Classes & Beans](../records-classes-and-beans/).

## Serialize and deserialize

The companion exposes four static methods — string and `byte[]` variants of `toJSON` and `fromJSON`:

```java
import com.example.internal.UserJSON;

User user = new User("alice", 37, "alice@example.com");

// Serialize
String json  = UserJSON.toJSON(user);       // String
byte[] bytes = UserJSON.toJSONBytes(user);   // byte[]

// Deserialize
User fromString = UserJSON.fromJSON(json);   // from String
User fromBytes  = UserJSON.fromJSON(bytes);  // from byte[]
```

`json` is the resulting wire form:

```json
{
  "name": "alice",
  "age": 37,
  "email": "alice@example.com"
}
```

There's no reflection anywhere on this path — the companion is plain generated Java that reads and writes the fields directly.

## Where the companion lives

The companion is generated in an `internal` subpackage of the annotated type's package, with the class name `<SimpleName>JSON`:

| Annotated type        | Generated companion                  |
|-----------------------|--------------------------------------|
| `com.example.User`    | `com.example.internal.UserJSON`      |
| `com.example.api.Order` | `com.example.api.internal.OrderJSON` |

Import the companion from that `internal` package wherever you serialize or deserialize.

## Next steps

- [Supported Types](../supported-types/) — what you can put in a `@JSON` type
- [Field Customization](../field-customization/) — rename keys, omit fields, and control output
- [Polymorphism](../polymorphism/) — sealed hierarchies with a discriminator
