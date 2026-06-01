---
layout: docs
title: Exception Handling
description: Render exceptions into HTTP responses with the ExceptionHandler middleware and HTTPException types.
weight: 100
---

## The pattern

Throw exceptions where your code wants to bail out, and let an `ExceptionHandler` middleware turn them into HTTP responses. Each exception type is mapped to an `ErrorRenderer` — a function that owns the entire response for that exception: it sets the status code **and** writes the body.

~~~~ java
import module java.base;
import module org.lattejava.web;

class NoteNotFoundException extends RuntimeException {}

void main() {
  var errors = new ExceptionHandler(Map.of(
      NoteNotFoundException.class, (req, res, e) -> {
        res.setStatus(404);
        res.getWriter().write("{\"error\":\"not found\"}");
      }
  ));

  new Web()
      .install(errors)
      .get("/notes/{id}", (req, res) -> {
        // ... look up the note, throw NoteNotFoundException if missing ...
      })
      .start(8080);
}
~~~~

`ErrorRenderer` is a functional interface:

~~~~ java
void render(HTTPRequest req, HTTPResponse res, Exception e) throws Exception;
~~~~

The middleware walks the thrown exception's class hierarchy from most specific to most general, looking for a registered renderer. The first match wins.

## Resolution order

When an exception propagates up to the `ExceptionHandler`, it is resolved in this order:

1. If a renderer is registered for the exception's class (or any of its supertypes), that renderer handles it.
2. Otherwise, if the exception is an `HTTPException` (see below), the **default renderer** handles it.
3. Otherwise, the exception is re-thrown — unexpected exceptions surface as the server's 500 rather than being silently swallowed.

## HTTPException

`HTTPException` is the base class for exceptions that carry an HTTP status code. Throw it (or one of its subtypes) anywhere and the framework will render it without any per-type registration:

~~~~ java
throw new HTTPException(409, "That username is taken");
~~~~

Latte ships semantic subtypes for the common cases, all in the `org.lattejava.web` package:

| Exception | Status |
|---|---|
| `BadRequestException` | 400 |
| `UnauthenticatedException` | 401 |
| `ForbiddenException` | 403 |
| `ServiceUnavailableException` | 503 |

Each has a no-arg, a `(String message)`, and a `(String message, Throwable cause)` constructor. The OIDC middlewares throw `UnauthenticatedException`, `ForbiddenException`, and `ServiceUnavailableException`; the JSON body supplier throws `BadRequestException` when a body fails to parse.

### The default renderer

`ExceptionHandler.DEFAULT_RENDERER` outputs JSON. It reads the status from the exception (the `HTTPException` status, or `500` otherwise) and, when the exception has a message, writes a body of the form:

~~~~ json
{"error":"BadRequestException", "message":"The request body could not be parsed as JSON"}
~~~~

Even when no `ExceptionHandler` is installed, `Web` renders any uncaught `HTTPException` with this default renderer as a baseline safety net — so a malformed JSON body produces a `400` out of the box.

## Customizing the default renderer

Pass your own default renderer to handle every `HTTPException` consistently — for example, to emit HTML instead of JSON:

~~~~ java
ErrorRenderer htmlDefault = (req, res, e) -> {
  int status = (e instanceof HTTPException he) ? he.status() : 500;
  res.setStatus(status);
  res.setContentType("text/html; charset=utf-8");
  res.getWriter().write("<h1>" + status + "</h1>");
};

// default renderer + per-type overrides:
var errors = new ExceptionHandler(htmlDefault, Map.of(
    NoteNotFoundException.class, (req, res, e) -> { res.setStatus(404); /* ... */ }
));
~~~~

The four `ExceptionHandler` constructors are:

| Constructor | Behavior |
|---|---|
| `ExceptionHandler()` | Default renderer, no per-type renderers. |
| `ExceptionHandler(ErrorRenderer defaultRenderer)` | Custom default renderer, no per-type renderers. |
| `ExceptionHandler(Map<Class<? extends Throwable>, ErrorRenderer>)` | Default renderer plus per-type renderers. |
| `ExceptionHandler(ErrorRenderer, Map<Class<? extends Throwable>, ErrorRenderer>)` | Both. |

To change how renderers are resolved (rather than which renderers exist), subclass and override `protected ErrorRenderer resolveRenderer(Class<?> type)`.

## Where to install

`ExceptionHandler` can only catch exceptions thrown by middlewares and handlers **below** it in the chain. For the broadest coverage, install it early — typically right after `SecurityHeaders` and `OriginChecks`:

~~~~ java
web.install(SecurityHeaders.defaults());
web.install(new OriginChecks());
web.install(errors);
// ... OIDC session endpoints, protected routes, etc.
~~~~

That way exceptions from your handlers, the OIDC middlewares, or any per-route middlewares all flow through the same renderers.
