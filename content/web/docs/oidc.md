---
layout: docs
title: OIDC Authentication
description: OpenID Connect login, callback, refresh, logout, and role checks.
weight: 110
---

## Overview

The `org.lattejava.web.oidc` package provides drop-in OpenID Connect authentication. It works against any standards-compliant OIDC provider, but we recommend [FusionAuth](https://fusionauth.io/) — it runs locally during development (so you don't need to depend on a third-party service), deploys self-hosted in production, and is the most robust of the OIDC providers we've tested. It's also what `web` itself runs against in its own OIDC integration test suite. Keycloak, Auth0, Okta, Google, and other compliant providers also work.

You configure it once with `OIDCConfig`, install an `OIDC` instance globally to handle the OAuth2 flow paths, then attach `oidc.authenticated()` (or a role check) to anything you want protected.

## Configuration

Build a config with the issuer URL and your client credentials:

~~~~ java
import module org.lattejava.web;

var config = OIDCConfig.builder()
    .issuer("https://auth.example.com")
    .clientId("my-app")
    .clientSecret(System.getenv("OIDC_SECRET"))
    .build();
~~~~

The framework discovers the authorize, token, userinfo, jwks, and (optional) logout endpoints from the issuer's `/.well-known/openid-configuration` document. To bypass discovery, set the endpoints explicitly:

~~~~ java
var config = OIDCConfig.builder()
    .clientId("my-app")
    .clientSecret(System.getenv("OIDC_SECRET"))
    .authorizeEndpoint(URI.create("https://auth.example.com/oauth2/authorize"))
    .tokenEndpoint(URI.create("https://auth.example.com/oauth2/token"))
    .userinfoEndpoint(URI.create("https://auth.example.com/oauth2/userinfo"))
    .jwksEndpoint(URI.create("https://auth.example.com/.well-known/jwks.json"))
    .build();
~~~~

Default scopes are `openid`, `profile`, `email`, and `offline_access` (so refresh tokens are issued). Override with `.scopes(List.of(...))` if your provider expects something different.

### Default paths

| Path | Purpose | Override with |
|---|---|---|
| `/login` | Initiates the OIDC flow | `.loginPath(String)` |
| `/oidc/return` | Authorization code callback | `.callbackPath(String)` |
| `/logout` | Initiates RP-initiated logout | `.logoutPath(String)` |
| `/oidc/logout-return` | Post-logout redirect target | `.logoutReturnPath(String)` |

After login the user is redirected to `postLoginPage` (default `/`). After logout, to `postLogoutPage` (default `/`).

### Cookies

Tokens are stored in `SameSite=Strict` cookies (default names `access_token`, `refresh_token`, `id_token`; OIDC state in `oidc_state`; the post-login destination in `oidc_return_to`). Refresh tokens have a 30-day max age by default.

## Wiring it up

`OIDC.create(config)` returns an `OIDC<JWT>` — the `<JWT>` is the type returned by `oidc.user()`. Install the instance globally so it owns the OIDC paths, then protect any subset of routes:

~~~~ java
import module org.lattejava.http;
import module org.lattejava.jwt;
import module org.lattejava.web;

void main() {
  var config = OIDCConfig.builder()
      .issuer("https://auth.example.com")
      .clientId("my-app")
      .clientSecret(System.getenv("OIDC_SECRET"))
      .build();

  var oidc = OIDC.create(config);

  try (var web = new Web()) {
    web.install(new SecurityHeaders());
    web.install(new OriginChecks());
    web.install(oidc);

    web.get("/", (req, res) -> res.getWriter().write("public landing"));

    web.prefix("/app", app -> {
      app.install(oidc.authenticated());
      app.get("/me", (req, res) -> {
        JWT jwt = OIDC.jwt();
        res.getWriter().write("Hi " + jwt.subject);
      });
    });

    web.start(8080);
  }
}
~~~~

The `OIDC` middleware itself only handles the four OIDC paths. Every other path passes through to the rest of your chain.

## Accessing the user

Inside a protected handler, `OIDC.jwt()` returns the bound JWT, or throws `UnauthenticatedException` if no JWT is present. (The exception maps to `401` automatically when an `ExceptionHandler` is installed.)

~~~~ java
app.get("/me", (req, res) -> {
  JWT jwt = OIDC.jwt();
  String email = jwt.getString("email");
  res.getWriter().write("hello " + email);
});
~~~~

If you'd rather not throw, use `OIDC.optionalJWT()`:

~~~~ java
web.get("/maybe", (req, res) -> {
  String greeting = OIDC.optionalJWT()
      .map(jwt -> "hello " + jwt.subject)
      .orElse("hello stranger");
  res.getWriter().write(greeting);
});
~~~~

## Translating the JWT to a domain object

The two-arg `OIDC.create` accepts a `Function<JWT, U>` that maps the JWT to your own user type. The translator is applied lazily on every call to `oidc.user()`.

~~~~ java
record AppUser(String id, String email, Set<String> roles) {}

Function<JWT, AppUser> translator = jwt -> new AppUser(
    jwt.subject,
    jwt.getString("email"),
    new HashSet<>(jwt.getList("roles", String.class))
);

OIDC<AppUser> oidc = OIDC.create(config, translator);

// Inside a handler:
AppUser user = oidc.user();
~~~~

`oidc.optionalUser()` is the safe equivalent that returns `Optional<U>`.

## Role-based access

If your JWT carries a `roles` claim (the default `roleExtractor` reads `jwt.getList("roles", String.class)`), you can require roles directly:

~~~~ java
web.prefix("/admin", admin -> {
  admin.install(oidc.authenticated());
  admin.install(oidc.hasAnyRole("admin", "superuser"));
  admin.get("/users", listAllUsers);
  admin.delete("/users/{id}", deleteUser, oidc.hasAllRoles("admin"));
});
~~~~

- `hasAnyRole(String...)` lets the request through if the user has **at least one** of the listed roles
- `hasAllRoles(String...)` requires **every** listed role

Both middlewares respond with `401` if no JWT is bound and `403` if the JWT is bound but the role check fails. They depend on `authenticated()` having run earlier in the chain.

If your provider exposes roles under a different claim, supply a custom extractor:

~~~~ java
var config = OIDCConfig.builder()
    .issuer("https://auth.example.com")
    .clientId("my-app")
    .clientSecret(secret)
    .roleExtractor(jwt -> Set.copyOf(jwt.getList("groups", String.class)))
    .build();
~~~~

## Hooking up your app

Once you've configured `OIDCConfig` and installed the `OIDC` instance on your `Web` (see [Wiring it up](#wiring-it-up)), the entire login and logout flow is just two links in your UI:

~~~~ html
<a href="/login">Sign in</a>
<a href="/logout">Sign out</a>
~~~~

That's it — no JavaScript, no state to manage, no callback handler to write. The `OIDC` middleware owns both paths. Clicking **Sign in** redirects the browser through the provider, lands on `/oidc/return`, sets the token cookies, and finally redirects to `postLoginPage`. Clicking **Sign out** clears the cookies, calls the provider's end-session endpoint, and redirects to `postLogoutPage`.

A typical landing page that swaps the button based on auth state:

~~~~ java
web.get("/", (req, res) -> {
  res.setContentType("text/html; charset=utf-8");
  String button = OIDC.optionalJWT()
      .map(jwt -> "<a href=\"/logout\">Sign out " + jwt.subject + "</a>")
      .orElse("<a href=\"/login\">Sign in</a>");
  res.getWriter().write("<h1>Welcome</h1>" + button);
});
~~~~

If you want login/logout to land somewhere other than `/`, set `postLoginPage` and `postLogoutPage` on the config:

~~~~ java
var config = OIDCConfig.builder()
    .issuer("https://auth.example.com")
    .clientId("my-app")
    .clientSecret(secret)
    .postLoginPage("/dashboard")
    .postLogoutPage("/goodbye")
    .build();
~~~~

Token refresh happens transparently — when `Authenticated` sees an expired access token, it uses the refresh token to mint a new one before the request continues. If refresh fails, the user is redirected to `/login` to start over.

## Putting it together

See the [Sample Application](../sample-application/) page for a complete server that uses OIDC, role checks, JSON bodies, static files, and security headers together.
