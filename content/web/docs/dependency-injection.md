---
layout: docs
title: Dependency Injection
description: Building handlers, controllers, and middleware with any dependency injection framework.
weight: 62
---

## Overview

Latte web has no dependency injection container of its own. Instead, `Web` accepts an `Injector`, a one-method interface that any DI library can satisfy, and uses it to build handlers, controllers, and middleware on demand:

~~~~ java
@FunctionalInterface
public interface Injector {
  <T> T get(Class<T> type);
}
~~~~

Register the injector once, then wrap types with `web.inject(...)` wherever a `Handler`, `BodyHandler`, or `Middleware` is expected. The framework calls the injector on every request, so your DI framework controls instance lifetime: a singleton is resolved once and reused, while a prototype is built fresh for each request.

The examples on this page use [Avaje Inject](https://avaje.io/inject/), which is what the framework's own tests use, but nothing here is specific to it. Any container with a "get an instance by class" method works, including Guice, Dagger, Spring, or a hand-written `Map<Class<?>, Object>`.

## Registering an injector

Call `injector(...)` on the root `Web` before registering routes that use `inject`. A method reference to the container's lookup method is usually enough:

~~~~ java
import module org.lattejava.web;

import io.avaje.inject.BeanScope;

void main() {
  try (var scope = BeanScope.builder().build(); var web = new Web()) {
    web.injector(scope::get)
       .get("/users/{id}", web.inject(UserHandler.class, UserHandler::handle))
       .start(8080);
  }
}
~~~~

The injector is inherited by every `prefix(...)` child, so `api.inject(...)` inside a prefix block resolves from the same container. It must be set on the root `Web` (not a prefix child) and before `start(...)`.

## Injecting handlers

The two-argument `inject(type, method)` resolves an instance of `type` from the injector on each request and invokes `method` on it. The method must accept the request and response after the instance itself, so a class that implements `Handler` fits directly:

~~~~ java
import module org.lattejava.http;
import module org.lattejava.web;

import jakarta.inject.Singleton;

@Singleton
public class UserHandler implements Handler {
  private final UserService users;

  public UserHandler(UserService users) {
    this.users = users;
  }

  @Override
  public void handle(HTTPRequest req, HTTPResponse res) throws Exception {
    String name = users.find((String) req.getAttribute("id"));
    if (name == null) {
      res.setStatus(404);
      return;
    }

    res.setStatus(200);
    res.getWriter().write("Hello, " + name);
  }
}
~~~~

~~~~ java
web.get("/users/{id}", web.inject(UserHandler.class, UserHandler::handle));
~~~~

## Injecting controllers

The handler class does not need to implement `Handler`. Any method on any class with the signature `(HTTPRequest, HTTPResponse)` can be bound, which lets one controller hold several related handlers:

~~~~ java
import module org.lattejava.http;

import io.avaje.inject.Prototype;

@Prototype
public class UserController {
  private final UserService users;

  public UserController(UserService users) {
    this.users = users;
  }

  public void show(HTTPRequest req, HTTPResponse res) throws Exception {
    String name = users.find((String) req.getAttribute("id"));
    res.getWriter().write("Hello, " + name);
  }

  public void create(HTTPRequest req, HTTPResponse res, NewUser body) {
    users.save(body.id(), body.name());
    res.setStatus(201);
  }
}
~~~~

~~~~ java
web.injector(scope::get)
   .get("/users/{id}", web.inject(UserController.class, UserController::show))
   .post("/users", web.inject(UserController.class, UserController::create),
         BodySupplier.of(NewUserJSON::fromJSON));
~~~~

The functional interfaces behind the two `inject` overloads are:

~~~~ java
@FunctionalInterface
public interface ControllerHandler<C> {
  void handle(C controller, HTTPRequest req, HTTPResponse res) throws Exception;
}

@FunctionalInterface
public interface ControllerBodyHandler<C, T> {
  void handle(C controller, HTTPRequest req, HTTPResponse res, T body) throws Exception;
}
~~~~

Because the controller Class is the first parameter, an unbound method reference such as `UserController::show` matches. The compiler figures everything out, including when a third parameter is the body object.

The `@Prototype` annotation above is an Avaje scope, which means that each request will use a new instance of the `UserController`. If this is changed to `Singleton`, then the same instance will be used each time. Each dependency injection framework will handle these concepts differently. Refer to the docs for the dependency injection framework you intend to use.

## Injecting middleware

The one-argument `inject(type)` returns a `Middleware` that resolves an instance of `type` on each request and delegates to it. The type must implement `Middleware`:

~~~~ java
import module org.lattejava.http;
import module org.lattejava.web;

import jakarta.inject.Singleton;

@Singleton
public class AuditMiddleware implements Middleware {
  private final AuditLog log;

  public AuditMiddleware(AuditLog log) {
    this.log = log;
  }

  @Override
  public void handle(HTTPRequest req, HTTPResponse res, MiddlewareChain chain) throws Exception {
    log.record(req.getPath());
    chain.next(req, res);
  }
}
~~~~

The result is an ordinary middleware, so it can be installed at any of the three scopes described in [Middleware](../middleware/):

~~~~ java
// Global
web.install(web.inject(AuditMiddleware.class));

// Prefix
web.prefix("/admin", admin -> admin.install(admin.inject(AuditMiddleware.class)));

// Per-route
web.post("/users", createUser, BodySupplier.of(NewUserJSON::fromJSON), web.inject(AuditMiddleware.class));
~~~~

## Instance lifetime

`inject` never caches. Each request calls `injector.get(type)`, and whatever comes back is used for that request only. The table below shows what that means for common scopes:

| Container scope               | Instances created                         |
|-------------------------------|-------------------------------------------|
| Singleton                     | One, on first request (or at startup)     |
| Prototype / unscoped          | One per request                           |
| Request-scoped (if supported) | One per request, managed by the container |

If the container itself is expensive to query, resolve the instance once and register it directly instead:

~~~~ java
UserHandler handler = injector.get(UserHandler.class);
web.get("/users/{id}", handler);
~~~~

`inject` is for the cases where you want the container to manage the instances for every request.