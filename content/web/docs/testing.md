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

**`JSONBodyAsserter`** (in `org.lattejava.web.test.json`) parses the body as JSON and asserts over the tree, either element by element or against a checked-in expected file. It has its own section: [JSON assertions](#json-assertions).

### Resetting between requests

`WebTestAsserter.reset(...)` clears state and returns the parent `WebTest` so you can chain the next request. `reset()` with no arguments clears both the cookie jar and the pending request state. The overload `reset(ResetItem... items)` lets you choose what to clear: `ResetItem.Cookies`, `ResetItem.Request`, and `ResetItem.HttpClient` (which closes and replaces the underlying client).

~~~~ java
test.get("/hello")
    .assertStatus(200)
    .reset()                 // fresh cookies + request state
    .get("/goodbye")
    .assertStatus(200);
~~~~

## JSON assertions

`JSONBodyAsserter` lives in `org.lattejava.web.test.json`. It parses the response body with the Latte `json` library and holds values in their natural Java shapes (`Map`, `List`, `String`, `Long`, `BigInteger`, `BigDecimal`, `Boolean`, and `null`). Element paths are [RFC 6901 JSON Pointers](https://www.rfc-editor.org/rfc/rfc6901): `""` is the root, `/user/name` is a property, `/items/0/id` indexes into an array, and a `/` inside a property name is escaped as `~1`.

Two rules apply to every comparison:

- Object keys are unordered, per JSON semantics.
- Types are strict. The number `1` does not match `1.0`, and the number `33` does not match the string `"33"`.

Arrays are compared as unordered multisets by default (`[1, 2, 3]` matches `[3, 2, 1]`, but `[1, 1, 2]` does not match `[1, 2, 2]`). Pass `false` to the constructor or call `unorderedArrays(false)` for positional comparison. The setting can be changed between assertions on the same instance.

### Elements and values

- `hasElement(String pointer)` / `hasNoElement(String pointer)` — assert a node exists or not.
- `hasValue(String pointer, String expected)` — compares the value's text form, so the JSON `33` matches `"33"` here.
- `hasValue(String pointer, Object expected)` — strictly typed comparison. The expected value is converted to its JSON shape the same way the `json` library would serialize it. A missing pointer is treated as `null`, so `hasValue("/missing", null)` passes.

~~~~ java
import org.lattejava.web.test.json.*;

var json = new JSONBodyAsserter();
test.withHeader("Accept", "application/json")
    .get("/api/users/42")
    .assertStatus(200)
    .assertBodyAs(json, j -> j
        .hasValue("/id", 42)
        .hasValue("/name", "Alice")
        .hasElement("/roles")
        .hasNoElement("/password")
    );
~~~~

### Whole-document equality

- `equalTo(String json)` — the body must be JSON-equivalent to the given text. Whitespace and key order are ignored.
- `equalTo(Object value)` — the body must equal the JSON form of a Java value. `Map`s become objects, `Iterable`s and arrays become arrays, enums use `name()`, and `UUID`, `URI`, `URL`, and the `java.time` types use their string forms. Records and POJOs with public fields are converted by reflection, with `null` components omitted. In a named module, the record's package must be opened to `org.lattejava.web`.
- `equalTo(Function<byte[], T> converter, T expected)` — parses the body with your own function, typically a generated `fromJSON` method, and compares with `equals`.

~~~~ java
test.get("/api/users/42")
    .assertBodyAs(new JSONBodyAsserter(), j -> j.equalTo("""
        {"id": 42, "name": "Alice", "roles": ["admin", "user"]}
        """));

test.get("/api/users/42")
    .assertBodyAs(new JSONBodyAsserter(), j -> j.equalTo(UserJSON::fromJSON, expectedUser));
~~~~

Comparing a response against the same `@JSON` object that produced it is circular: a change to the object changes both sides, so a wire-format break still passes. For public API tests, prefer an expected file.

### Expected files

`equalToFile(Path file, Object... substitutions)` compares the body against a JSON file committed to the repo. Any change to the wire format then shows up as a diff to a checked-in file. The convention is to keep the files under `src/test/json`, referenced by `Path`:

~~~~ java
test.get("/api/users/42")
    .assertStatus(200)
    .assertBodyAs(new JSONBodyAsserter(), j ->
        j.equalToFile(Paths.get("src/test/json/users/get.json")));
~~~~

`src/test/json/users/get.json`:

~~~~ json
{
  "id": 42,
  "name": "Alice",
  "roles": ["admin", "user"]
}
~~~~

The comparison is strict in both directions: an extra field, a missing field, or a type change all fail. Object keys are unordered, but **arrays are always positional** because order is part of the wire format. `equalToFile` ignores the `unorderedArrays` setting.

On failure, both documents are pretty-printed so the IDE diff shows only the semantic change.

### Substitutions

A test often knows a value that changes from run to run, such as an ID it created or the port the server is on. Reference it in the file as `${name}` and supply the value as a name/value pair after the path:

~~~~ json
{
  "id": "${id}",
  "count": "${count}",
  "active": "${active}",
  "self": "http://localhost:${port}/users/${id}"
}
~~~~

~~~~ java
j.equalToFile(Paths.get("src/test/json/users/get.json"),
    "id", user.id(), "count", 42, "active", true, "port", 8080);
~~~~

A string node that is exactly one token takes the JSON type of the supplied value. `"count": "${count}"` becomes the number `42`, and `"active": "${active}"` becomes the boolean `true`. Keeping the token inside quotes keeps the file valid JSON for editors. A string node with an embedded token, like the `self` URL above, splices in the value's text form and stays a string.

Substitution happens on the parsed tree, never by text templating, so a value containing quotes or braces cannot break the document.

Two mistakes fail the assertion: a `${...}` token in the file that names nothing you supplied, and a supplied substitution the file never references. Both are there to catch typos and stale tests. To put a literal `${` in the expected text, escape it as `$${`.

### Placeholders

For values the test cannot know, such as a generated UUID or a timestamp, use a placeholder. A placeholder must be the entire string node:

| Placeholder        | Matches                                                     |
|--------------------|-------------------------------------------------------------|
| `${anyBoolean}`    | Any boolean                                                 |
| `${anyInstant}`    | A string accepted by `Instant.parse` (ISO 8601, not epoch millis) |
| `${anyNumber}`     | Any JSON number                                             |
| `${anyString}`     | Any string                                                  |
| `${anyUUID}`       | A string accepted by `UUID.fromString`                      |
| `${regex:PATTERN}` | A scalar whose text form fully matches `PATTERN`            |

~~~~ json
{
  "id": "${anyUUID}",
  "created": "${anyInstant}",
  "version": "${regex:\\d+\\.\\d+\\.\\d+}",
  "name": "Alice"
}
~~~~

A placeholder asserts the type at its position. The key must still be present, and a placeholder never matches a JSON `null`.

`anyNumber`, `anyString`, and `anyInstant` accept an inclusive range after a colon. Either bound may be left empty for an open end, but not both:

| Token                                                        | Constraint                        |
|--------------------------------------------------------------|-----------------------------------|
| `${anyNumber:[-10:100]}`                                     | Value between -10 and 100         |
| `${anyNumber:[0:]}`                                          | Value at least 0                  |
| `${anyString:[2:5]}`                                         | Length between 2 and 5            |
| `${anyInstant:[2026-01-01T00:00:00Z/2026-12-31T00:00:00Z]}`  | Instant within the interval       |
| `${anyInstant:[2026-01-01T00:00:00Z/]}`                      | Instant on or after the start     |

Instant ranges use a slash separator, following ISO 8601 interval notation, because instants contain colons. The `regex` pattern runs to the closing brace at the very end of the node, so it may contain colons and braces of its own.

### Generating and updating files

You do not have to write expected files by hand:

- **Missing file:** the file is generated from the actual response, pretty-printed with a two-space indent in wire key order, and the assertion fails with a reminder to review and commit it. Replace any run-specific values with substitutions or placeholders before committing.
- **Mismatch with `-Dlatte.web.json.update=true`:** the file is rewritten from the actual response and the assertion passes. Tokens that still describe the actual value are preserved, so a `${id}` whose value matches stays `${id}`, and a `${anyInstant}` that still matches stays a placeholder. Everything else is written literally. Review the result with `git diff`.
- **In CI:** files are never written. A missing file fails the test, and setting the update property fails the test on its own. CI is detected when the `CI` environment variable is set and non-empty. Override the protected `ci()` method in a subclass to change the detection.

Set the property for a single run, for example as a VM option in your IDE's run configuration, or temporarily through the [TestNG plugin's](../../../cli/docs/plugin-java-testng/#jvm-arguments) JVM arguments in `project.latte`:

~~~~ groovy
javaTestNG.settings.jvmArguments = "-Dlatte.web.json.update=true"
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
import org.lattejava.web.test.json.*;

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
