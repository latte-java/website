---
layout: docs
title: Sample Application
description: A complete server that uses every major feature together.
weight: 120
---

## What it does

This sample wires together:

- A public landing page
- Static asset serving
- An OIDC-protected `/app` section
- A JSON API behind authentication and a role check
- Security headers, CSRF defense, and exception-to-status mapping

It's everything in the docs in one file.

## `Main.java`

~~~~ java
import module com.fasterxml.jackson.databind;
import module org.lattejava.http;
import module org.lattejava.jwt;
import module org.lattejava.web;

void main() {
  // --- OIDC -----------------------------------------------------------------
  var oidcConfig = OIDCConfig.builder()
                             .issuer("https://auth.example.com")
                             .clientId("notes-app")
                             .clientSecret(System.getenv("OIDC_SECRET"))
                             .postLoginPage("/app")
                             .postLogoutPage("/")
                             .build();
  var oidc = OIDC.create(oidcConfig);

  // --- Exception -> status --------------------------------------------------
  var errors = new ExceptionHandler(
      Map.of(
          NoteNotFoundException.class, 404
      )
  );

  var web = new Web();
  web.baseDir(Paths.get("web"));

  // --- Global middlewares -------------------------------------------------
  web.install(new SecurityHeaders());
  web.install(new OriginChecks());
  web.install(errors);
  web.install(oidc);

  // --- Static assets ------------------------------------------------------
  web.files("/static");

  // --- Public routes ------------------------------------------------------
  web.get("/", (_, res) -> {
    res.setContentType("text/html; charset=utf-8");
    res.getWriter().write("""
        <!doctype html>
        <html><head><title>Notes</title>
        <link rel="stylesheet" href="/static/app.css"></head>
        <body>
          <h1>Notes</h1>
          <a href="/login">Sign in</a>
        </body></html>
        """);
  });

  web.get("/health", (_, res) -> res.setStatus(200));

  // --- Authenticated app --------------------------------------------------
  web.prefix("/app", app -> {
    app.install(oidc.authenticated());

    app.get("/", (_, res) -> {
      JWT jwt = OIDC.jwt();
      res.setContentType("text/html; charset=utf-8");
      res.getWriter().write("<h1>Welcome, " + jwt.subject() + "</h1>");
    });
  });

  // --- Authenticated JSON API --------------------------------------------
  web.prefix("/api", api -> {
    api.install(oidc.authenticated());

    api.get("/notes/{id}", (req, res) -> {
      String id = (String) req.getAttribute("id");
      // throws NoteNotFoundException -> 404 via ExceptionHandler
      Note note = NoteStore.find(id);
      writeJson(res, 200, note);
    });

    api.post("/notes",
        (_, res, note) -> {
          JWT jwt = OIDC.jwt();
          Note created = NoteStore.create(jwt.subject(), note);
          writeJson(res, 201, created);
        },
        JSONBodySupplier.of(NewNote.class),
        oidc.hasAnyRole("editor", "admin"));

    api.delete("/notes/{id}",
        (req, res) -> {
          String id = (String) req.getAttribute("id");
          NoteStore.delete(id);
          res.setStatus(204);
        },
        oidc.hasAllRoles("admin"));
  });

  web.start(8080);
  System.out.println("Listening on http://localhost:8080");
}

private static void writeJson(HTTPResponse res, int status, Object body) throws Exception {
  res.setStatus(status);
  res.setContentType("application/json");
  res.getWriter().write(new ObjectMapper().writeValueAsString(body));
}

record NewNote(String title, String body) {
}

record Note(String id, String owner, String title, String body) {
}

static class NoteNotFoundException extends RuntimeException {
}

static class NoteStore {
  private static final Map<String, Note> store = new ConcurrentHashMap<>();

  static Note create(String owner, NewNote input) {
    String id = UUID.randomUUID().toString();
    Note n = new Note(id, owner, input.title(), input.body());
    store.put(id, n);
    return n;
  }

  static void delete(String id) {
    store.remove(id);
  }

  static Note find(String id) {
    Note n = store.get(id);
    if (n == null) throw new NoteNotFoundException();
    return n;
  }
}
~~~~

## What's happening

A few details worth calling out:

- `web.baseDir(Paths.get("web"))` points the static-file middleware at the `web/` directory the project template creates. With `web.files("/static")`, requests for `/static/app.css` are served from `web/static/app.css`
- The middleware install order is **headers, then CSRF, then exceptions, then OIDC**. That means CSRF rejection happens before any auth or business logic runs, and any exception thrown after that point is mapped to a status code by the time it reaches the HTTP server
- `oidc.hasAnyRole(...)` and `oidc.hasAllRoles(...)` are passed as **per-route** middlewares, after the body supplier. They run after the prefix-level `oidc.authenticated()` so the JWT is already bound when the role check executes
- `OIDC.jwt()` is the static accessor that returns the JWT bound to the current request. It throws `UnauthenticatedException` (mapped to 401) if called outside a protected route — handy as a sanity check
- `Web` is not closed, which leaves the server running. The framework also registers a JVM shutdown hook on `start`, so `Ctrl-C` works

## Running it

With the snippet above saved as `src/main/java/Main.java`:

~~~~ bash
OIDC_SECRET=...
latte run
~~~~

**NOTE**: This is just an example and is not connected to a real OIDC provider. You would need to connect the example to a IdP such as FusionAuth.
