---
layout: docs
title: Records, Classes & Beans
description: The three kinds of @JSON types and how each is read and written.
weight: 30
---

`@JSON` can be placed on three kinds of types. All three generate the same companion API ([`toJSON` / `fromJSON`](../getting-started/#serialize-and-deserialize)); they differ only in how the processor reads values out for serialization and constructs an instance on deserialization.

## Records

A record is the simplest case: its components are the members, read through the record accessors and written through the canonical constructor. No extra annotations are required.

```java
@JSON
public record User(String name, int age, String email) {}
```

## Classes with `@JSONConstructor`

For an immutable (non-record) class, mark the constructor the deserializer should call with `@JSONConstructor`. The constructor's **parameter names** become the members.

```java
@JSON
public class Point {
  private final int x;
  private final int y;

  @JSONConstructor
  public Point(int x, int y) {
    this.x = x;
    this.y = y;
  }

  public int getX() { return x; }
  public int getY() { return y; }
}
```

`{"x":1,"y":2}` deserializes by calling `new Point(1, 2)` and serializes by reading `getX()` / `getY()`.

**Reading values back out.** For a parameter named `foo`, the processor resolves a reader in this order: `getFoo()`, then `isFoo()` (boolean only), then `foo()`, then a public field `foo`. If a serialized parameter has no usable reader, compilation fails with a message telling you to add one (or mark the parameter write-only).

**Rules:**

- Exactly one `@JSONConstructor`, and it must be `public` (the companion lives in a sibling `internal` package and has to reach it).
- `@JSONConstructor` is not valid on a record.
- A parameter marked [`@JSONField(writeOnly = true)`](../field-customization/) needs no reader — useful for inputs like a password that should never be echoed back out.

## JavaBean classes

A `@JSON` class **without** a `@JSONConstructor` is treated as a JavaBean: the processor constructs it with the public no-arg constructor and discovers properties from accessors.

```java
@JSON
public class Account {
  private String id;
  private int balance;

  public String getId() { return id; }
  public void setId(String id) { this.id = id; }

  public int getBalance() { return balance; }
  public void setBalance(int balance) { this.balance = balance; }

  public int getFeeBps() { return balance > 100 ? 0 : 25; }  // computed, read-only
}
```

**Property discovery.** Properties come from `getFoo()` / `isFoo()` (read), `setFoo(value)` (write), and public fields. A property that has only a reader is **read-only** (serialized but not deserialized); one with only a writer is **write-only**. So `getFeeBps()` above is serialized but never read back — the normal pattern for a computed value.

**Rules:**

- A public no-arg constructor is required.
- At least one property must be discovered.
- `static` members are ignored, and `transient` fields are skipped.
- When both a field and an accessor for the same property carry [`@JSONField`](../field-customization/), the field's annotation wins.

## Polymorphic subtypes

Records, `@JSONConstructor` classes, and JavaBeans can all be the permitted subtypes of a sealed `@JSON` hierarchy — see [Polymorphism](../polymorphism/).
