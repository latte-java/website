---
layout: docs
title: Middleware
description: The middleware pipeline, ordering, and writing your own.
weight: 60
---

## The interface

A middleware is a `@FunctionalInterface`:

~~~~ java
@FunctionalInterface
public interface Middleware {
  void handle(HTTPRequest req, HTTPResponse res, MiddlewareChain chain) throws Exception;
}
~~~~

It can do three things:

1. Inspect or mutate the request and response
2. Pass control downstream by calling `chain.next(req, res)`
3. Short-circuit by **not** calling `chain.next` (for example, write a 401 and return)

Anything you do **after** `chain.next(...)` runs as the chain unwinds — useful for timing, response logging, or post-processing headers.

## Installing middleware

There are three scopes:

**Global** — runs for every request. Install on the root `Web`:

~~~~ java
web.install(new SecurityHeaders());
web.install(new OriginChecks());
~~~~

**Prefix** — runs only for requests that match the prefix:

~~~~ java
web.prefix("/api", api -> {
  api.install(rateLimit);
  api.get("/users", listUsers);
});
~~~~

**Per-route** — passed as trailing varargs to a verb method:

~~~~ java
web.post("/admin/wipe", wipeHandler, requireAdmin, requireConfirmation);
~~~~

## Execution order

For a matched request, middlewares run in this order:

1. Global middlewares, in registration order
2. Prefix middlewares, from outermost prefix to innermost
3. Per-route middlewares, in registration order
4. The handler
5. Each middleware's post-`next` code, in **reverse** of the above

So `chain.next(...)` is your "now do everything below me" call, and any code after it runs on the way back out.

## Writing a request logger

A common middleware that times every request:

~~~~ java
Middleware logger = (req, res, chain) -> {
  long start = System.currentTimeMillis();
  chain.next(req, res);
  long elapsed = System.currentTimeMillis() - start;
  System.out.printf("%s %s -> %d (%dms)%n",
      req.getMethod().name(), req.getPath(), res.getStatus(), elapsed);
};

web.install(logger);
~~~~

## Writing an auth gate

Short-circuit a request by setting a status and returning without calling `chain.next`:

~~~~ java
Middleware requireApiKey = (req, res, chain) -> {
  String key = req.getHeader("X-API-Key");
  if (key == null || !key.equals(System.getenv("API_KEY"))) {
    res.setStatus(401);
    return;
  }
  chain.next(req, res);
};

web.prefix("/api", api -> {
  api.install(requireApiKey);
  api.get("/users", listUsers);
});
~~~~

## Mutating the response after the handler runs

Because middlewares unwind in reverse, you can adjust the response after the handler is done:

~~~~ java
Middleware addRequestId = (req, res, chain) -> {
  String id = UUID.randomUUID().toString();
  req.setAttribute("requestId", id);
  chain.next(req, res);
  res.setHeader("X-Request-Id", id);
};
~~~~

> **Warning:** This only works if the response hasn't been flushed yet. Once any downstream middleware or handler writes to `res.getWriter()` or `res.getOutputStream()` and the bytes leave the server (or the response is explicitly flushed), the headers and status are committed and any subsequent `setHeader` / `setStatus` call is a no-op. If you need a header to be present, set it **before** calling `chain.next(...)`, or make sure no downstream code flushes early.

## Built-in middleware

The framework ships several middlewares you'll use in production:

- [`SecurityHeaders`](../security-headers/) — strict default response headers
- [`OriginChecks`](../csrf-protection/) — CSRF defense via `Origin` validation
- [`StaticResources`](../static-files/) — serves files from disk
- [`ExceptionHandler`](../exception-handling/) — maps exceptions to status codes
- [`OIDC`](../oidc/) — OpenID Connect login, callback, refresh, and logout

Each is documented on its own page.
