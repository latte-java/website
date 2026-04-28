---
layout: docs
title: Handlers
description: Writing request handlers and using HTTPRequest and HTTPResponse.
weight: 40
---

## The `Handler` interface

A handler is a `@FunctionalInterface` with a single method:

~~~~ java
@FunctionalInterface
public interface Handler {
  void handle(HTTPRequest req, HTTPResponse res) throws Exception;
}
~~~~

That means handlers can be lambdas, method references, or full classes. The simplest possible one returns a status:

~~~~ java
web.get("/health", (req, res) -> res.setStatus(200));
~~~~

Handlers may throw any exception. Anything not caught by a downstream middleware (such as the [`ExceptionHandler`](../exception-handling/)) propagates out and the underlying HTTP server returns a 500.

## Reading the request

`HTTPRequest` lives in `org.lattejava.http`. Useful methods:

- `getPath()` and `getMethod()` for routing decisions
- `getHeader(name)` and `getHeaders(name)` for headers
- `getCookies()` for incoming cookies
- `getInputStream()` for the raw body bytes (use a [`BodySupplier`](../request-bodies/) for parsing)
- `getAttribute(name)` for path parameters and any data middleware has stashed
- `getBaseURL()` for `scheme://host:port` (handy when building absolute URLs)

Path parameters declared with `{name}` syntax are exposed as request attributes:

~~~~ java
web.get("/orders/{id}", (req, res) -> {
  String id = (String) req.getAttribute("id");
  // ...
});
~~~~

Reading a header:

~~~~ java
String acceptLang = req.getHeader("Accept-Language");
~~~~

## Writing the response

`HTTPResponse` provides:

- `setStatus(int)` for the status code
- `setHeader(name, value)` and `setContentType(String)` for headers
- `getWriter()` for text bodies (a `PrintWriter`)
- `getOutputStream()` for binary bodies
- `sendRedirect(String)` for 302 redirects
- `addCookie(Cookie)` for `Set-Cookie`

A handler that writes plain text:

~~~~ java
web.get("/hello", (req, res) -> {
  res.setStatus(200);
  res.setContentType("text/plain; charset=utf-8");
  res.getWriter().write("Hello, world!");
});
~~~~

A handler that writes JSON manually (for richer JSON responses, see the [Request Bodies](../request-bodies/) page):

~~~~ java
web.get("/api/health", (req, res) -> {
  res.setStatus(200);
  res.setContentType("application/json");
  res.getWriter().write("{\"status\":\"ok\"}");
});
~~~~

A redirect:

~~~~ java
web.get("/old-path", (req, res) -> res.sendRedirect("/new-path"));
~~~~

## Reusing handlers as method references

Because `Handler` is a functional interface, any method with a matching signature can be used as a handler:

~~~~ java
public class UsersController {
  public void list(HTTPRequest req, HTTPResponse res) throws Exception {
    res.getWriter().write("users");
  }

  public void show(HTTPRequest req, HTTPResponse res) throws Exception {
    res.getWriter().write("user " + req.getAttribute("id"));
  }
}

void main() {
  var users = new UsersController();
  new Web()
      .get("/users", users::list)
      .get("/users/{id}", users::show)
      .start(8080);
}
~~~~

This is a useful pattern when handlers need shared collaborators (a database connection, a service, etc.) — construct the controller once with its dependencies, then bind methods to routes.
