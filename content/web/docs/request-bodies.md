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

The supplier is invoked first, and whatever it returns — including `null` — is passed straight to the handler. A `null` therefore means "an empty but valid body," and the handler is given the chance to deal with it. A supplier signals a *failure* by **throwing** an exception (typically an `HTTPException` subtype such as `BadRequestException`), not by returning `null`. See [Exception Handling](../exception-handling/) for how those exceptions become responses.

This pattern keeps parsing concerns out of your handler: validation, content-type checking, and error reporting live in the supplier.

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

If the JSON is malformed or doesn't match the target type, `JSONBodySupplier` throws `BadRequestException`, which the framework renders as `400 Bad Request` (even without an installed `ExceptionHandler`). An empty request body returns `null`, and your handler is invoked with that `null`. To return a richer error payload, register a renderer for `BadRequestException` (see [Exception Handling](../exception-handling/)) or write your own supplier.

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

A more interesting one validates while it parses, throwing to reject a bad body:

~~~~ java
BodySupplier<NewUser> validatedSupplier = (req, res) -> {
  NewUser user = new ObjectMapper().readValue(req.getInputStream(), NewUser.class);
  if (user.email() == null || !user.email().contains("@")) {
    throw new HTTPException(422, "invalid email");
  }
  return user;
};
~~~~

By throwing, the supplier short-circuits the request without your handler having to know about validation; the exception is turned into a response by the [exception handling](../exception-handling/) machinery.

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
