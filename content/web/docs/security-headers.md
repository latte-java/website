---
layout: docs
title: Security Headers
description: Strict default response headers via the SecurityHeaders middleware.
weight: 80
---

## Why

Browsers respect a handful of HTTP response headers that mitigate clickjacking, MIME sniffing, mixed content, and a long tail of other attacks. The right defaults are well understood, so the framework ships them as a single middleware: `SecurityHeaders`.

## Defaults

Install with no arguments to apply every default:

~~~~ java
import module org.lattejava.web;

web.install(new SecurityHeaders());
~~~~

The middleware writes its headers **before** invoking the rest of the chain, so they appear on every response — including 404s and 5xx errors. A handler that calls `setHeader` on the same name overrides the default.

The headers and their default values:

| Header | Default |
|---|---|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` |
| `Content-Security-Policy` | A strict default |
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` |
| `X-XSS-Protection` | `0` (the modern recommendation) |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | A locked-down default |
| `Cross-Origin-Opener-Policy` | `same-origin` |
| `Cross-Origin-Embedder-Policy` | `require-corp` |
| `Cross-Origin-Resource-Policy` | `same-origin` |

## Customizing

Use the builder to override individual headers. Pass `null` to suppress one entirely:

~~~~ java
web.install(SecurityHeaders.builder()
    .contentSecurityPolicy("default-src 'self'; img-src 'self' data:; script-src 'self'")
    .strictTransportSecurity("max-age=63072000; includeSubDomains; preload")
    .crossOriginEmbedderPolicy(null)   // disable COEP for compatibility
    .build());
~~~~

## Where it doesn't run

`405 Method Not Allowed` responses bypass the middleware chain (they're produced by the router before middleware kicks in), so they only carry the `Allow` header — no security headers. They also have no body, so the missing headers aren't a meaningful gap. Every other response — including 404 — runs through the middleware and gets the full set.

## Recommendation

Install `SecurityHeaders` early in your middleware list, with `OriginChecks` right after it:

~~~~ java
web.install(new SecurityHeaders());
web.install(new OriginChecks());
~~~~

That order means the headers are set first; later middlewares (or handlers) can still override them where needed.
