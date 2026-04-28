---
layout: docs
title: Static Files
description: Serving CSS, JavaScript, and other assets with caching and traversal protection.
weight: 70
---

## The shorthand

The fastest way to serve static files is `Web.files(urlPrefix)`:

~~~~ java
web.files("/assets");
~~~~

That installs a `StaticResources` middleware that maps `/assets/*` to `<baseDir>/assets/*` on disk. The base directory defaults to the current working directory; override it with `Web.baseDir(Path)` if needed:

~~~~ java
web.baseDir(Paths.get("/var/www")).files("/assets");
~~~~

If the URL prefix and the on-disk subdirectory don't match, pass both:

~~~~ java
web.files("/assets", "public");
// /assets/app.css -> <baseDir>/public/app.css
~~~~

## The middleware

Under the hood, `files` installs a `StaticResources` middleware. Use the constructor directly when you need control over caching or want to filter requests:

~~~~ java
import module java.base;
import module org.lattejava.web;

void main() {
  var assets = new StaticResources(
      "/assets",
      "public",
      Duration.ofDays(30),
      null);

  new Web()
      .baseDir(Paths.get("/var/www"))
      .install(assets)
      .start(8080);
}
~~~~

What the middleware gives you:

- **Content-Type inference** based on file extension
- **`ETag`, `Last-Modified`, and `304 Not Modified`** handling for conditional requests
- **`Cache-Control` and `Expires`** headers driven by the `cacheDuration` argument (default 7 days)
- **Path traversal protection** — paths that resolve outside the base directory are rejected. The underlying `HTTPContext.resolve(String)` returns `null` for any escape attempt and the middleware responds with a 404

## Combining with routes

A `StaticResources` middleware only handles requests under its URL prefix. Other paths fall through to the rest of the chain, so you can mix static and dynamic routes freely:

~~~~ java
web.files("/assets")
   .get("/", indexHandler)
   .get("/api/users/{id}", showUser);
~~~~

Requests to `/assets/app.css` are served from disk; everything else hits your handlers.

## Layout for the `web` template

The `latte init web` template creates a `web/static/` directory at the project root and sets the working directory accordingly. To serve everything in that directory under `/static`:

~~~~ java
web.baseDir(Paths.get("web")).files("/static");
~~~~

Now `web/static/app.css` is reachable at `http://localhost:8080/static/app.css`.
