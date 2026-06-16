---
title: "json"
description: "A small, hardened JSON library for Java 25 with compile-time codegen — no reflection, nothing on your runtime path"
layout: landing
github: "https://github.com/latte-java/json"
---

A small, hardened JSON library for Java 25. Annotate a record, class, or sealed interface with `@JSON` and a compile-time annotation processor generates the serializer and deserializer for it — no reflection, and nothing from this library on your runtime path.

## Install

Add the processor to your `project.latte`:

```groovy
dependencies {
  group(name: "compile-processors") {
    dependency(id: "org.lattejava:json:0.4.0")
  }
}
```

## Example

```java
@JSON
public record User(String name, int age, String email) {}
```

```java
import com.example.internal.UserJSON;

String json = UserJSON.toJSON(user);     // serialize
User   back = UserJSON.fromJSON(json);   // deserialize
```

## Features

- **Fast** — throughput at or above Jackson with far less allocation ([benchmarks](docs/performance/))
- **Compile-time codegen** — zero reflection; the companion is plain generated Java
- **No runtime dependency** — `org.lattejava:json` never lands on your compile classpath or runtime path
- **Records, classes, and JavaBeans** — plus discriminated sealed hierarchies
- **Flexible mapping** — naming strategies, per-field renames, catch-all members, arbitrarily nested collections, and date/time control

[Read the documentation →](docs/)

## License

MIT
