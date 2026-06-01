---
layout: docs
title: Testing
description: A fluent HTTP test client and an OIDC fixture for driving a running Web app from your tests.
weight: 115
---

## Overview

The `org.lattejava.web.test` package gives you two things for testing a Latte Web application:

- **`WebTest`** — a fluent HTTP client that issues real requests against a `Web` instance running on a port and returns an asserter for the response.
- **`OIDCTestFixture`** — a helper that walks the full OAuth2 authorization-code + PKCE flow against a real IdP so your tests can run as an authenticated user.

Because `WebTest` makes real HTTP calls over `localhost`, you start your `Web` app exactly as you would in production, then point a `WebTest` at the same port. There is no mock layer — what you test is what runs.

## Starting a Web app for tests

Start the app on a port and create a `WebTest` for that port. `Web` implements `AutoCloseable`, so a try-with-resources block tears the server down at the end of the test.

~~~~ java
import module org.lattejava.web;

import org.lattejava.web.test.*;

void example() {
  try (var web = new Web()) {
    web.get("/hello", (_, res) -> {
      res.setContentType("text/plain");
      res.getWriter().write("Hello, world!");
    });
    web.start(8080);

    var test = new WebTest(8080);
    test.get("/hello")
        .assertStatus(200)
        .assertBodyAs(new StringBodyAsserter(), b -> b.equalTo("Hello, world!"));
  }
}
~~~~

`new WebTest(int port)` builds an `HttpClient` configured for tests: a short connect timeout, a virtual-thread executor, and—importantly—**redirects are never followed**. That last point lets you assert directly on `3xx` responses and their `Location` headers instead of chasing them.

## Configuring a request

A `WebTest` is configured with chained `with*` methods, then fired with a verb method. The verb methods are `get`, `post`, `put`, `patch`, `delete`, `head`, and `options`, each taking a path and returning a `WebTestAsserter`.

~~~~ java
test.withHeader("Accept", "application/json")
    .withURLParameter("page", "2")
    .get("/api/users");
~~~~

The builder methods:

- `withHeader(String name, String value)` — adds a request header. Multiple values for the same name are allowed and sent in registration order.
- `withURLParameter(String name, String value)` — adds a query-string parameter. Repeated names are supported.
- `withBody(byte[] body)` / `withBody(String body)` — sets the raw request body. The `String` overload encodes as UTF-8.
- `withForm(Map<String, String> form)` — sets form fields from a map (iteration order preserved).
- `withFormField(String name, String value)` — appends a single form field. Duplicate names are preserved and sent in registration order.
- `withCookie(Cookie cookie)` / `withCookie(String name, String value)` — adds a cookie to the jar (see below).

### Bodies vs. forms

`withBody*` and `withForm*` are mutually exclusive—**the most recent call wins.** Setting a body clears any form fields you registered, and registering a form field (or a form map) clears any body you set. They never coexist on a request.

When form fields are present, they are encoded as `application/x-www-form-urlencoded` and used as the request body. The `Content-Type` header is set to `application/x-www-form-urlencoded` automatically **unless you already supplied one** with `withHeader`. If you supply a `Content-Type` that is *not* `application/x-www-form-urlencoded` while using the form methods, the request throws an `IllegalStateException`.

~~~~ java
test.withFormField("email", "user@example.com")
    .withFormField("password", "secret")
    .post("/login")
    .assertStatus(302);
~~~~

## The cookie jar

`WebTest` exposes a public `cookies` field of type `CookieJar` that **persists across requests**. Cookies set by the server via `Set-Cookie` are parsed and stored after every response, then sent back automatically on subsequent requests—so a login that sets a session cookie just works for the calls that follow.

The jar is backed by `org.lattejava.http.Cookie` rather than `java.net.HttpCookie`, which means it preserves the `SameSite` attribute that the JDK's cookie store silently drops. It honors deletions too: a `Set-Cookie` with `Max-Age=0` or an expired `Expires` removes the cookie from the jar.

`WebTest` also exposes its other request state as public fields: `port`, `body` (the raw body bytes), `headers`, `formFields`, and `urlParameters`.

## Assertions

Each verb call returns a `WebTestAsserter`. Its assertion methods are chainable and throw `AssertionError` on failure with messages formatted to match TestNG's wire format (so IDE comparison-failure highlighting works).

- `assertStatus(int expected)` — asserts the response status code.
- `assertHeader(String name, String expected)` — asserts the first value of a response header equals `expected`.
- `assertHeaderStartsWith(String name, String expected)` — asserts the first value starts with `expected`.
- `assertRedirect(int expectedStatus, String expectedLocation)` — convenience for `assertStatus` + `assertHeader("Location", ...)`.
- `assertCookie(String name, String value)` — asserts the jar holds a cookie with the given name and value.
- `assertResponse(Consumer<HttpResponse<byte[]>>)` — runs an arbitrary assertion against the raw response.
- `assertBodyAs(T bodyAsserter, Consumer<T> consumer)` — feeds the response body into a `BodyAsserter` and runs the consumer's assertions.
- `response()` — returns the underlying `HttpResponse<byte[]>` for direct access.

### Asserting on bodies

`assertBodyAs` takes a `BodyAsserter` and a consumer. Two implementations ship in the box.

**`StringBodyAsserter`** treats the body as a UTF-8 string:

- `equalTo(String)` / `notEqualTo(String)`
- `contains(String)` / `doesNotContain(String)`
- `matches(String regex)` — full match
- `isEmpty()` / `isNotEmpty()`

~~~~ java
var string = new StringBodyAsserter()
test.get("/hello")
    .assertStatus(200)
    .assertBodyAs(string, s -> s.contains("Hello"));
~~~~

**`JSONBodyAsserter`** parses the body as JSON and asserts over the tree using [RFC 6901 JSON Pointers](https://www.rfc-editor.org/rfc/rfc6901):

- `equalTo(String json)` / `equalTo(Object value)` — whole-document equality. Object keys are always unordered; arrays default to unordered (multiset) comparison.
- `hasElement(String pointer)` / `hasNoElement(String pointer)` — assert a node exists or not.
- `hasValue(String pointer, String expected)` — text-value comparison (the JSON `33` matches `"33"`).
- `hasValue(String pointer, Object expected)` — strictly typed comparison via `ObjectMapper.valueToTree`.
- `unorderedArrays(boolean)` — toggle multiset vs. positional array comparison (also a constructor argument).

~~~~ java
var json = new JSONBodyAsserter();
test.withHeader("Accept", "application/json")
    .get("/api/users/42")
    .assertStatus(200)
    .assertBodyAs(json, j -> j
        .hasValue("/id", 42)
        .hasValue("/name", "Alice")
        .hasElement("/roles")
    );
~~~~

### Resetting between requests

`WebTestAsserter.reset(...)` clears state and returns the parent `WebTest` so you can chain the next request. `reset()` with no arguments clears both the cookie jar and the pending request state. The overload `reset(ResetItem... items)` lets you choose what to clear: `ResetItem.Cookies`, `ResetItem.Request`, and `ResetItem.HttpClient` (which closes and replaces the underlying client).

~~~~ java
test.get("/hello")
    .assertStatus(200)
    .reset()                 // fresh cookies + request state
    .get("/goodbye")
    .assertStatus(200);
~~~~

## Authenticating with `OIDCTestFixture`

`OIDCTestFixture` drives the full OAuth2 authorization-code flow (with PKCE) against a real identity provider—for example a running FusionAuth—so your tests execute as a logged-in user. It submits the hosted-login form, follows the redirect chain to your registered redirect URI, captures the authorization code, exchanges it at the token endpoint, and stores the issued tokens as cookies in the `WebTest`'s cookie jar under the default names `access_token`, `refresh_token`, and `id_token`. After `login` returns, every subsequent request through that `WebTest` is authenticated.

Construct a fixture with a `WebTest` and an `OIDCConfig`. An overload accepts a `BrowserSettings` to control cookie names and paths:

~~~~ java
var fixture = new OIDCTestFixture(test, oidcConfig);
// or, with explicit browser settings:
var fixture = new OIDCTestFixture(test, oidcConfig, BrowserSettings.builder().build());
~~~~

One fixture represents one OAuth client. The client is identified by `OIDCConfig.clientId()` on every request, and the token-exchange shape follows `OIDCConfig.publicClient()`:

- **Confidential clients** (`publicClient=false`, the default) authenticate at the token endpoint with the configured `clientSecret`.
- **Public clients** (`publicClient=true` — CLI, native, desktop, console) send `client_id` in the form body and rely on PKCE.

PKCE is used in both modes.

### Logging in and out

- `login(String email, String password)` — SSR convenience. Defaults the redirect URI to `http://localhost:<port><browser.callbackPath()>`.
- `login(String email, String password, String redirectURI)` — supplies an explicit redirect URI, for flows whose registered redirect is a loopback (`http://127.0.0.1:PORT/...`) or a custom scheme (`myapp://callback`).
- `logout()` — removes the access, refresh, and id token cookies from the cookie jar, so subsequent requests are unauthenticated.

Both `login` overloads return a `Tokens` record (`accessToken`, `refreshToken`, `idToken`, `expiresIn`)—any field may be `null` if the IdP omits it. The tokens are also stored in the cookie jar, so you usually don't need the return value unless you want the raw tokens (for example to set an `Authorization: Bearer` header in a CLI simulation).

### End-to-end example

~~~~ java
import module org.lattejava.web;

import org.lattejava.web.oidc.*;
import org.lattejava.web.test.*;

void protectedRouteTest() throws Exception {
  var oidcConfig = OIDCConfig.builder()
                             .issuer("https://auth.example.com")
                             .clientId("notes-app")
                             .clientSecret(System.getenv("OIDC_SECRET"))
                             .build();

  var oidc = OIDC.ssr(oidcConfig);

  try (var web = new Web()) {
    web.install(OIDC.sessionEndpoints(oidcConfig));
    web.prefix("/app", app -> {
      app.install(oidc.authenticated());
      app.get("/profile", (_, res) -> {
        res.setContentType("application/json");
        res.getWriter().write("{\"email\":\"user@example.com\"}");
      });
    });
    web.start(9011);

    var test = new WebTest(9011);

    // Unauthenticated requests are challenged.
    test.get("/app/profile")
        .assertStatus(302)
        .reset();

    // Log in against the real IdP — tokens land in the cookie jar.
    var fixture = new OIDCTestFixture(test, oidcConfig);
    Tokens tokens = fixture.login("user@example.com", "password");

    // Now the same test client is authenticated.
    test.get("/app/profile")
        .assertStatus(200)
        .assertBodyAs(new JSONBodyAsserter(), json ->
            json.hasValue("/email", "user@example.com"));

    fixture.logout();
  }
}
~~~~

The IdP is assumed to already be running and configured to accept the client identified by `clientId`. There is no `applicationId` concept—the client is identified solely by `clientId`, and its credentials are dictated by `publicClient`.
