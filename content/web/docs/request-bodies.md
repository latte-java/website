---
layout: docs
title: Request Bodies
description: Parsing request bodies with BodySupplier and BodyHandler.
weight: 50
---

## The pattern

Reading the request body in a handler is two steps: parse it into something useful, then hand it to your handler. Latte web breaks those into two interfaces:

~~~~ java
@FunctionalInterface
public interface BodySupplier<T> {
  T get(HTTPRequest req, HTTPResponse res) throws Exception;
}

@FunctionalInterface
public interface BodyHandler<T> {
  void handle(HTTPRequest req, HTTPResponse res, T body) throws Exception;
}
~~~~

You wire them together by registering the route with a body-aware overload:

~~~~ java
web.post("/users", createUserHandler, JSONBodySupplier.of(NewUser.class));
~~~~

The supplier is invoked first. If it returns `null` it has signalled a handled error condition (it set the status, etc.) and the handler is **not** called. If it returns a value, that value is passed to the handler.

This pattern keeps parsing concerns out of your handler: validation, content-type checking, and error responses live in the supplier.

## JSON bodies

The framework ships with `JSONBodySupplier`, a Jackson-backed implementation. The shortest path:

~~~~ java
import module com.fasterxml.jackson.databind;
import module org.lattejava.http;
import module org.lattejava.web;

record NewUser(String email, String name) {}

void main() {
  new Web()
      .post("/users",
            (req, res, user) -> {
              res.setStatus(201);
              res.setContentType("application/json");
              res.getWriter().write("{\"created\":\"" + user.email() + "\"}");
            },
            JSONBodySupplier.of(NewUser.class))
      .start(8080);
}
~~~~

If the JSON is malformed or doesn't match the target type, `JSONBodySupplier` sets the response status to `400 Bad Request`, returns `null`, and the handler is skipped. To return a richer error payload, subclass `JSONBodySupplier` or write your own supplier.

You can also pass a custom `ObjectMapper` (for example, to register modules):

~~~~ java
ObjectMapper mapper = new ObjectMapper().findAndRegisterModules();
web.post("/users", handler, JSONBodySupplier.of(NewUser.class, mapper));
~~~~

## Custom suppliers

A supplier is just a function from request to value. Here's one that reads the body as plain text:

~~~~ java
BodySupplier<String> stringSupplier = (req, res) -> {
  byte[] bytes = req.getInputStream().readAllBytes();
  return new String(bytes, StandardCharsets.UTF_8);
};

web.post("/echo", (req, res, body) -> res.getWriter().write(body), stringSupplier);
~~~~

A more interesting one validates while it parses:

~~~~ java
BodySupplier<NewUser> validatedSupplier = (req, res) -> {
  NewUser user = new ObjectMapper().readValue(req.getInputStream(), NewUser.class);
  if (user.email() == null || !user.email().contains("@")) {
    res.setStatus(422);
    res.setContentType("application/json");
    res.getWriter().write("{\"error\":\"invalid email\"}");
    return null;
  }
  return user;
};
~~~~

By returning `null` after writing a response, the supplier short-circuits the request without your handler having to know about validation.

## When you don't need a supplier

If you're happy reading the raw stream yourself (a file upload, a streaming endpoint, a non-JSON binary format), use the regular `Handler` overload and call `req.getInputStream()` directly:

~~~~ java
web.post("/upload", (req, res) -> {
  Path target = Paths.get("/var/uploads", UUID.randomUUID() + ".bin");
  try (var out = Files.newOutputStream(target)) {
    req.getInputStream().transferTo(out);
  }
  res.setStatus(201);
});
~~~~
