---
layout: docs
title: Cookies
description: Reading, writing, clearing, and optionally encrypting cookies with secure-by-default attributes.
weight: 55
---

## Overview

`Cookies` is an application-facing helper for reading, writing, and clearing cookies. It gives every cookie secure defaults and can optionally encrypt values with AES-256-GCM authenticated encryption. Build one instance at application startup and reuse it across requests and threads — it is immutable and thread-safe.

There are two factory methods:

~~~~ java
import module org.lattejava.web;

// No encryption — plain cookies only
var cookies = Cookies.newInstance();

// With encryption keys (enables the .encrypted() variants)
var cookies = Cookies.encryptionKeys(key);
~~~~

## Writing a cookie

Start with `write(name, value)`, chain any options, then commit with `to(req, res)`:

~~~~ java
cookies.write("theme", "dark")
       .maxAge(Duration.ofDays(365))
       .to(req, res);
~~~~

The chainable options on the write builder are:

- `path(String)` — defaults to `/`
- `domain(String)` — unset by default
- `httpOnly(boolean)` — defaults to `true`
- `sameSite(Cookie.SameSite)` — defaults to `Cookie.SameSite.Strict` (`Lax`, `None`, `Strict`)
- `maxAge(Duration)` — unset by default (a session cookie); stored as whole seconds
- `secure(boolean)` — when not set, derived from the request scheme (see below)

The request is passed to `to(...)` so the helper can auto-derive the `Secure` attribute.

## Secure by default

Every cookie the helper writes (and every cookie it clears) starts from a hardened baseline:

- `HttpOnly` is `true`
- `SameSite` is `Strict`
- `Secure` is auto-derived: it is set when the request scheme is `https`, or when the `X-Forwarded-Proto` header is `https` (for requests behind a TLS-terminating proxy)

Calling `secure(true)` or `secure(false)` explicitly overrides the auto-derivation; otherwise the helper picks the right value per request.

## Reading a cookie

`read(name).from(req)` returns the cookie value as a `String`, or `null` if the cookie is absent:

~~~~ java
String theme = cookies.read("theme").from(req);
if (theme == null) {
  theme = "light";
}
~~~~

## Clearing a cookie

`clear(name).from(req, res)` writes an expired cookie (max-age `0`) so the browser drops it. Match the `path` (and `domain`, if any) you used when writing, otherwise the browser keeps the original cookie:

~~~~ java
cookies.clear("theme").from(req, res);

// With a custom path/domain
cookies.clear("session")
       .path("/app")
       .domain("example.com")
       .from(req, res);
~~~~

The clear builder supports `path(String)` and `domain(String)`. The cleared cookie carries the same secure defaults (`HttpOnly`, `SameSite=Strict`, auto-derived `Secure`).

## Encrypted cookies

If the helper was built with `Cookies.encryptionKeys(...)`, call `.encrypted()` on the read or write builder to encrypt and decrypt the value transparently. Values are protected with AES-256-GCM, and the cookie name is bound into the ciphertext as additional authenticated data, so a value encrypted under one name cannot be replayed under another.

~~~~ java
// Write — value is encrypted before it goes on the wire
cookies.write("uid", "user-12345")
       .encrypted()
       .maxAge(Duration.ofHours(8))
       .to(req, res);

// Read — value is decrypted and authenticated
String uid = cookies.read("uid").encrypted().from(req);
~~~~

The `encrypted()` builder is otherwise identical to the plain one — all the same `path`, `domain`, `httpOnly`, `sameSite`, `maxAge`, and `secure` options apply.

Calling `.encrypted()` on a helper created with `Cookies.newInstance()` throws `IllegalStateException`, because no keys are configured.

## Encryption keys and rotation

Keys must be AES-256 — exactly 32 bytes. You can supply them three ways:

~~~~ java
// Raw bytes (each must be 32 bytes)
var cookies = Cookies.encryptionKeys(rawKeyBytes);

// SecretKey instances
var cookies = Cookies.encryptionKeys(secretKey);

// A List<SecretKey>
var cookies = Cookies.encryptionKeys(List.of(newKey, oldKey));
~~~~

Pass more than one key to support rotation. Encryption always uses the **first** key in the list, while decryption tries **every** key in order. To rotate, prepend the new key and keep the old one until all cookies encrypted under it have expired:

~~~~ java
// Newly written cookies use newKey; existing cookies signed
// with oldKey still decrypt until they age out.
var cookies = Cookies.encryptionKeys(List.of(newKey, oldKey));
~~~~

All factory methods validate that each key is AES and 32 bytes, throwing `IllegalArgumentException` otherwise. Passing no keys is also rejected — use `Cookies.newInstance()` for the no-encryption case.

## Handling tampered cookies

When an encrypted cookie cannot be authenticated, reading it throws `CookieIntegrityException`. This happens either because the wire value is not valid Base64URL/AES-GCM data, or because the GCM tag fails to verify under every configured key. The latter is indistinguishable from active tampering — in both cases the correct response is to treat the cookie as invalid and clear it.

The exception exposes `name()` and `reason()`, where `reason()` is one of:

- `CookieIntegrityException.Reason.MALFORMED` — the value isn't well-formed ciphertext
- `CookieIntegrityException.Reason.DECRYPT_FAILED` — authentication failed under all keys

~~~~ java
String uid;
try {
  uid = cookies.read("uid").encrypted().from(req);
} catch (CookieIntegrityException e) {
  // Bad or tampered cookie — drop it and treat the user as logged out
  cookies.clear("uid").from(req, res);
  uid = null;
}
~~~~
