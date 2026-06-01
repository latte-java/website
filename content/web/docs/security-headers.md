---
layout: docs
title: Security Headers
description: Strict default response headers via the immutable SecurityHeaders middleware and the CSP builder.
weight: 80
---

## Why

Browsers respect a handful of HTTP response headers that mitigate clickjacking, MIME sniffing, mixed content, and a long tail of other attacks. The right defaults are well understood, so the framework ships them as a single middleware: `SecurityHeaders`.

## Defaults

`SecurityHeaders` is immutable. Start from one of two factory methods:

- `SecurityHeaders.defaults()` — every header set to its most-secure value.
- `SecurityHeaders.empty()` — no headers; opt in to the ones you want.

~~~~ java
import module org.lattejava.web;

web.install(SecurityHeaders.defaults());
~~~~

Each header is written **only if the response does not already have it** (`getHeader(name) == null`), so a handler or an upstream middleware can override any header simply by setting it first. The headers appear on every response that runs through the chain — including 404s and 5xx errors.

The headers and their default values:

| Header | Default |
|---|---|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains; preload` |
| `Content-Security-Policy` | `CSP.defaults().build()` (see below) |
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` |
| `X-XSS-Protection` | `0` (the modern recommendation) |
| `Referrer-Policy` | `no-referrer` |
| `Permissions-Policy` | `accelerometer=(), camera=(), geolocation=(), gyroscope=(), magnetometer=(), microphone=(), payment=(), usb=()` |
| `Cross-Origin-Opener-Policy` | `same-origin` |
| `Cross-Origin-Embedder-Policy` | `require-corp` |
| `Cross-Origin-Resource-Policy` | `same-origin` |

On `localhost` / `127.0.0.1` the middleware automatically strips `upgrade-insecure-requests` from the CSP so that local development over plain HTTP works.

## Customizing

Because the middleware is immutable, each per-header method returns a **new** instance with that one header changed. Chain them off `defaults()` (or `empty()`). Pass `null` to clear a header entirely:

~~~~ java
web.install(SecurityHeaders.defaults()
    .strictTransportSecurity("max-age=63072000; includeSubDomains; preload")
    .crossOriginEmbedderPolicy(null)            // disable COEP for compatibility
    .contentSecurityPolicy(CSP.defaults().addScriptSrc(CSP.SELF)));
~~~~

The per-header methods are: `contentSecurityPolicy(CSP)` / `contentSecurityPolicy(String)`, `strictTransportSecurity(String)`, `referrerPolicy(String)`, `permissionsPolicy(String)`, `crossOriginEmbedderPolicy(String)`, `crossOriginOpenerPolicy(String)`, `crossOriginResourcePolicy(String)`, `xContentTypeOptions(String)`, `xFrameOptions(String)`, and `xXSSProtection(String)`.

## The CSP builder

`CSP` builds a Content-Security-Policy string fluently. Start from `CSP.defaults()` (the policy used by `SecurityHeaders.defaults()`) or `CSP.empty()`, configure directives, and call `build()` (directives render in the order you add them).

`CSP.defaults()` produces:

~~~~
default-src 'self'; style-src 'self' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com; img-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'; form-action 'self'; upgrade-insecure-requests
~~~~

Each directive has a setter that **replaces** its sources (`defaultSrc`, `scriptSrc`, `styleSrc`, `imgSrc`, `fontSrc`, `connectSrc`, `frameSrc`, `frameAncestors`, `formAction`, `baseUri`, `objectSrc`, `mediaSrc`, `manifestSrc`, `childSrc`, `workerSrc`, `sandbox`, …), an `addX` variant that appends sources, and a `removeX` variant that removes them. There are also generic `directive(name, values...)`, `addDirective`, `removeDirective`, and `remove(name)` methods, the flag directive `upgradeInsecureRequests()`, and reporting helpers (`reportTo`, `reportUri`, `requireTrustedTypesFor`, `trustedTypes`).

Common source keywords are available as constants so you don't have to remember the quoting: `CSP.SELF`, `CSP.NONE`, `CSP.STRICT_DYNAMIC`, `CSP.UNSAFE_INLINE`, `CSP.UNSAFE_EVAL`, `CSP.UNSAFE_HASHES`, `CSP.WASM_UNSAFE_EVAL`, `CSP.REPORT_SAMPLE`, and `CSP.INLINE_SPECULATION_RULES`. The static helper `CSP.nonce(value)` formats a nonce source.

~~~~ java
var csp = CSP.defaults()
    .scriptSrc(CSP.SELF, CSP.STRICT_DYNAMIC)
    .addImgSrc("data:")
    .connectSrc(CSP.SELF, "https://api.example.com");

web.install(SecurityHeaders.defaults().contentSecurityPolicy(csp));
~~~~

## Where it doesn't run

`405 Method Not Allowed` responses bypass the middleware chain (they're produced by the router before middleware kicks in), so they only carry the `Allow` header — no security headers. They also have no body, so the missing headers aren't a meaningful gap. Every other response — including 404 — runs through the middleware and gets the full set.

## Recommendation

Install `SecurityHeaders` early in your middleware list, with `OriginChecks` right after it:

~~~~ java
web.install(SecurityHeaders.defaults());
web.install(new OriginChecks());
~~~~

That order means the headers are set first; later middlewares (or handlers) can still override them where needed.
