---
title: "web"
description: "A simple, yet powerful, web framework for Java"
layout: landing
github: "https://github.com/latte-java/web"
---

## Install

Scaffold a new project from the `web` template:

```bash
latte init web
```

Or add it to an existing Latte project:

```groovy
dependency(id: "org.lattejava:web:0.1.0")
```

## Quick Start

With Java 25 module imports and a class-less `void main`, a working server is a handful of lines:

```java
import module org.lattejava.http;
import module org.lattejava.web;

void main() {
  new Web()
      .get("/", (req, res) -> res.getWriter().write("Hello, world!"))
      .start(8080);
}
```

The fluent API supports all standard verbs, path parameters, and grouping with `prefix`:

```java
import module org.lattejava.http;
import module org.lattejava.web;

void main() {
  try (var web = new Web()) {
    web.get("/health", (req, res) -> res.setStatus(200))
       .prefix("/api", api -> {
         api.get("/users/{id}", (req, res) -> {
           String id = (String) req.getAttribute("id");
           res.getWriter().write("User " + id);
         });
         api.post("/users", (req, res) -> res.setStatus(201));
       })
       .start(8080);
  }
}
```

## Features

- **Fluent routing** - Register routes with `get`, `post`, `put`, `patch`, `delete`, `options`, or generic `route`
- **Path parameters** - Curly-brace syntax (`/users/{id}`); values are exposed as request attributes
- **Prefix groups** - Nestable `prefix(...)` blocks share a path and middleware scope
- **Middleware pipeline** - Global, prefix, and per-route middlewares run in registration order, then unwind in reverse
- **Body parsing** - Pluggable `BodySupplier<T>` interface; `JSONBodySupplier` is included for Jackson-backed JSON
- **OIDC authentication** - Drop-in `OIDC` middleware handles login, callback, refresh, logout, and role checks
- **Static files** - `StaticResources` serves files from disk with caching, ETag, and path-traversal protection
- **Security headers** - `SecurityHeaders` writes a strict default set; every header is overridable
- **CSRF defense** - `OriginChecks` rejects unsafe-method requests with a mismatched `Origin` header
- **Java 25 ready** - Built around module imports, the JPMS, and virtual-thread I/O via the Latte HTTP server

## Routing

Routes are registered against `Web`. Literal segments take priority over parameter segments, so `/users/new` and `/users/{id}` can coexist on the same verb.

```java
web.get("/users/new", (req, res) -> res.getWriter().write("new user form"))
   .get("/users/{id}", (req, res) -> {
     res.getWriter().write("user " + req.getAttribute("id"));
   });
```

When no route matches the path the framework returns **404**. When the path matches but the method doesn't, it returns **405** with an `Allow` header containing every method registered for that path.

## Middleware

Any lambda matching `(req, res, chain) -> ...` is a middleware. Call `chain.next(req, res)` to pass control downstream, or skip it to short-circuit.

```java
web.install((req, res, chain) -> {
  long start = System.currentTimeMillis();
  chain.next(req, res);
  System.out.printf("%s %s -> %d (%dms)%n",
      req.getMethod().name(), req.getPath(), res.getStatus(),
      System.currentTimeMillis() - start);
});
```

Per-route middlewares are passed as trailing varargs to any verb method:

```java
web.get("/admin", adminHandler, requireAdmin);
```

## Authentication

Wire up an OpenID Connect provider by installing an `OIDC` instance. It owns `/login`, `/oidc/return`, `/logout`, and `/oidc/logout-return` automatically. We recommend [FusionAuth](https://fusionauth.io/) — it runs locally for development, deploys self-hosted in production, and is the most robust of the OIDC providers we've tested. It's also what `web` itself uses for its OIDC integration tests. Any standards-compliant provider works (Keycloak, Auth0, Google, etc.).

```java
import module org.lattejava.http;
import module org.lattejava.jwt;
import module org.lattejava.web;

void main() {
  var config = OIDCConfig.builder()
      .issuer("https://auth.example.com")
      .clientId("my-app")
      .clientSecret(System.getenv("OIDC_SECRET"))
      .build();

  var oidc = OIDC.create(config);

  try (var web = new Web()) {
    web.install(oidc);
    web.prefix("/app", app -> {
      app.install(oidc.authenticated());
      app.get("/me", (req, res) -> {
        JWT jwt = OIDC.jwt();
        res.getWriter().write("Signed in as " + jwt.subject);
      });
    });
    web.start(8080);
  }
}
```

See the [documentation](docs/) for the full set of features, including JSON body parsing, role-based access, security headers, static files, and a complete sample application.
