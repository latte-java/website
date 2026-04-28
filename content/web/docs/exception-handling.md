---
layout: docs
title: Exception Handling
description: Mapping exceptions to HTTP status codes with the ExceptionHandler middleware.
weight: 100
---

## The pattern

Throw exceptions where your code wants to bail out. Install an `ExceptionHandler` middleware that maps exception types to HTTP status codes.

~~~~ java
import module java.base;
import module org.lattejava.web;

class NotFoundException extends RuntimeException {}
class ValidationException extends RuntimeException {
  ValidationException(String msg) { super(msg); }
}

void main() {
  var errors = new ExceptionHandler(Map.of(
      NotFoundException.class, 404,
      ValidationException.class, 422
  ));

  new Web()
      .install(errors)
      .get("/users/{id}", (req, res) -> {
        String id = (String) req.getAttribute("id");
        if (!id.matches("\\d+")) {
          throw new ValidationException("id must be numeric");
        }
        // ... lookup user, throw NotFoundException if missing ...
      })
      .start(8080);
}
~~~~

The middleware walks the exception's class hierarchy from most specific to most general. The first matching entry wins. If no entry matches, the exception propagates and the underlying HTTP server returns a 500.

## Built-in mappings

`UnauthenticatedException` (thrown by `OIDC.jwt()` and the OIDC middlewares when no JWT is bound) maps to `401` by default. You can override the mapping by including the class in the map you pass to the constructor.

## Writing an error body

`ExceptionHandler` only sets the status by default — no response body. To emit a JSON or HTML error payload, subclass and override `writeBody`:

~~~~ java
class JsonExceptionHandler extends ExceptionHandler {
  JsonExceptionHandler(Map<Class<? extends Throwable>, Integer> map) {
    super(map);
  }

  @Override
  protected void writeBody(HTTPRequest req, HTTPResponse res, Exception e, int status)
      throws Exception {
    res.setContentType("application/json");
    String message = e.getMessage() == null ? "" : e.getMessage().replace("\"", "\\\"");
    res.getWriter().write("{\"error\":\"" + message + "\",\"status\":" + status + "}");
  }
}

web.install(new JsonExceptionHandler(Map.of(
    NotFoundException.class, 404,
    ValidationException.class, 422
)));
~~~~

`writeBody` runs after the status has been set. Anything you throw from inside it propagates normally — there's no recursive catch.

## Where to install

`ExceptionHandler` works wherever in the chain you put it, but it can only catch exceptions thrown by middlewares **below** it. For the broadest coverage, install it early — typically right after `SecurityHeaders` and `OriginChecks`:

~~~~ java
web.install(new SecurityHeaders());
web.install(new OriginChecks());
web.install(new JsonExceptionHandler(...));
web.install(oidc);
// ... routes
~~~~

That way exceptions from your handlers, your OIDC middleware, or any per-route middlewares all flow through the same status mapping.
