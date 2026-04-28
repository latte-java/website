---
layout: docs
title: CSRF Protection
description: Defending mutating endpoints with the OriginChecks middleware.
weight: 90
---

## The strategy

Latte web uses two layers to defend against cross-site request forgery:

1. **`SameSite=Strict` cookies** — the underlying HTTP server sets session cookies with `SameSite=Strict` so browsers won't include them on cross-origin requests in the first place
2. **`Origin` header validation** — the `OriginChecks` middleware rejects unsafe-method requests whose `Origin` header doesn't match the server

There are no double-submit tokens or hidden form fields to manage. The cookies and the `Origin` header do the work together.

## Default install

The simplest install derives the allowed origin from the request itself:

~~~~ java
import module org.lattejava.web;

web.install(new OriginChecks());
~~~~

For each `POST`, `PUT`, `PATCH`, or `DELETE`, the middleware compares the `Origin` header to the request's own scheme + host + port. A match passes. A mismatch — or a missing `Origin` header on a browser request — gets rejected.

`GET`, `HEAD`, and `OPTIONS` requests are considered safe and pass through without an `Origin` check.

## Explicit allow-list

When your app is reachable under multiple origins (apex + www, multiple subdomains, a CDN), pass the full list explicitly:

~~~~ java
import module java.base;
import module org.lattejava.web;

web.install(new OriginChecks(List.of(
    URI.create("https://example.com"),
    URI.create("https://www.example.com"),
    URI.create("https://app.example.com")
)));
~~~~

A request whose `Origin` matches any URI in the list passes.

## When to install

`OriginChecks` should sit early in the global middleware chain, just after `SecurityHeaders`. Installing it once on the root `Web` covers every route, which is what you want — CSRF defense is a default-on protection, not a per-endpoint opt-in.

~~~~ java
web.install(new SecurityHeaders());
web.install(new OriginChecks());
web.install(oidc);
~~~~

## Limitations

- `OriginChecks` doesn't validate the `Referer` header. Modern browsers send `Origin` on all unsafe-method requests; if you also need to defend against very old clients you can layer a `Referer` check yourself
- It doesn't apply to safe methods. If you have a side-effecting `GET` (you shouldn't), make it a `POST`
- It doesn't replace authentication. CSRF protection only matters once the request also presents valid credentials
