---
layout: docs
title: Polymorphism
description: Discriminated sealed hierarchies with @JSONTypeInfo and @JSONSubtype.
weight: 80
---

A sealed `@JSON` interface can round-trip its permitted subtypes through a **discriminator** — a JSON key whose value names the concrete type. Annotate the sealed interface with `@JSONTypeInfo` to declare the discriminator key, and each permitted subtype with `@JSONSubtype` to declare its value.

```java
@JSON
@JSONTypeInfo(property = "petType")
public sealed interface Pet permits Dog, Cat, Bird {}

@JSON
public record Dog(String name, int packSize) implements Pet {}

@JSON
@JSONSubtype("kitty")
public record Cat(String name, int lives) implements Pet {}

@JSON
@JSONSubtype("Bird")
public record Bird(String name) implements Pet {}
```

The companion generated for the sealed type (`PetJSON`) has the same [`toJSON` / `fromJSON` API](../getting-started/#serialize-and-deserialize) as any other type, but it dispatches on the discriminator:

```java
Pet pet = PetJSON.fromJSON("{\"petType\":\"Dog\",\"name\":\"Rex\",\"packSize\":3}");  // a Dog
String json = PetJSON.toJSON(pet);
```

```json
{
  "petType": "Dog",
  "name": "Rex",
  "packSize": 3
}
```

## The discriminator

- `@JSONTypeInfo(property = "...")` names the discriminator key. It's required, and the annotated type must be a sealed interface that also carries `@JSON`.
- `@JSONSubtype("value")` names the value for a subtype. If you omit `@JSONSubtype` (or leave its value empty), the subtype's **simple class name** is used — so `Dog` above maps to `"Dog"`, while `Cat` is remapped to `"kitty"`.

On serialize, the discriminator is always written as the **first** key in the object. On deserialize, the parser reads the discriminator, constructs the matching subtype, and ignores the discriminator key when populating that subtype's members — so a subtype parses cleanly even under [`strict` mode](../field-customization/#output-options). A discriminator value that matches no subtype throws `JSONProcessingException`.

## Mixed subtype kinds

Permitted subtypes can be any mix of [records, `@JSONConstructor` classes, and JavaBeans](../records-classes-and-beans/) — each just needs its own `@JSON @JSONSubtype`. Polymorphic types also compose: a `Pet` can appear as a record component, a `List<Pet>`, or a `Map<String, Pet>`, and each element dispatches independently.

## Validation

The build fails (with a message pointing at the offending type) when:

- `@JSONTypeInfo` is on a non-sealed type, or a sealed `@JSON` interface is missing `@JSONTypeInfo`.
- a permitted subtype isn't annotated with `@JSON`, or `@JSONSubtype` appears on a type with no `@JSONTypeInfo` parent.
- two subtypes resolve to the same discriminator value.
- the discriminator key collides with a member name in any subtype.
