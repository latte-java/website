---
layout: docs
title: Routing
description: HTTP verbs, path parameters, prefixes, and how the router matches requests.
weight: 30
---

## Verbs

Every standard HTTP verb has a method on `Web`:

~~~~ java
web.get("/users", listUsers);
web.post("/users", createUser);
web.put("/users/{id}", replaceUser);
web.patch("/users/{id}", updateUser);
web.delete("/users/{id}", deleteUser);
web.options("/users", corsPreflight);
~~~~

To register a handler for several methods at once, use the generic `route`:

~~~~ java
web.route(List.of("GET", "HEAD"), "/health", (req, res) -> res.setStatus(200));
~~~~

All verb methods return the `Web` instance, so you can chain registrations.

## Path parameters

Use curly braces for path parameters. The matched value is stored on the request as an attribute keyed by the parameter name:

~~~~ java
web.get("/posts/{postId}/comments/{commentId}", (req, res) -> {
  String postId = (String) req.getAttribute("postId");
  String commentId = (String) req.getAttribute("commentId");
  res.getWriter().write("post=" + postId + " comment=" + commentId);
});
~~~~

Parameter names follow Java identifier rules (letters, digits, `_`, `$`; cannot start with a digit). Duplicate names within a single path are rejected at registration time.

Literal segments and parameter segments can coexist on the same path:

~~~~ java
web.get("/users/new", (req, res) -> res.getWriter().write("new user form"));
web.get("/users/{id}", (req, res) -> res.getWriter().write("user " + req.getAttribute("id")));
~~~~

A request to `/users/new` matches the literal route. A request to `/users/42` falls through to the parameterized one. Literal beats parameter at every segment, with no scoring or specificity rules to memorize.

## Prefix groups

`prefix(...)` scopes a block of registrations under a common path. Prefixes nest:

~~~~ java
web.prefix("/api", api -> {
  api.prefix("/v1", v1 -> {
    v1.get("/users", listUsers);          // GET /api/v1/users
    v1.post("/users", createUser);        // POST /api/v1/users
  });
  api.prefix("/v2", v2 -> {
    v2.get("/users", listUsersV2);        // GET /api/v2/users
  });
});
~~~~

Prefixes also scope **middleware** — anything you `install` inside a prefix block runs only for paths under that prefix. See the [Middleware](../middleware/) page.

## 404 vs 405

The router distinguishes "no route at this path" from "route exists but not for this method":

- **404 Not Found** - no route matches the request path
- **405 Method Not Allowed** - the path matches a registered route, but no handler is registered for the request method. The response includes an `Allow` header listing every method that *is* registered for that path, aggregated across literal and parameter branches

You don't write any code for these — `Web` handles them automatically.

## Lifecycle constraints

Routes and middleware can only be registered before `start(int)` is called. Attempting to register after the server has started throws `IllegalStateException`. The lock is shared across parent and child `Web` instances (the ones created by `prefix`), so once any `start` is called the entire registration tree is frozen.

`Web` implements `AutoCloseable`. Use try-with-resources for explicit lifetime control:

~~~~ java
try (var web = new Web()) {
  web.get("/", indexHandler);
  web.start(8080);
  // ...
}
~~~~

Or rely on the JVM shutdown hook that `start` registers automatically — the server will be closed cleanly on `Ctrl-C`.
