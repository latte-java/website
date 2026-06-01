---
layout: docs
title: OIDC Authentication
description: OpenID Connect for server-rendered apps, single-page apps, and APIs — login, refresh, logout, and role checks.
weight: 110
---

## Overview

The `org.lattejava.web.oidc` package provides drop-in OpenID Connect authentication. It works against any standards-compliant OIDC provider, but we recommend [FusionAuth](https://fusionauth.io/) — it runs locally during development, deploys self-hosted in production, and is what `web` itself runs against in its own OIDC integration test suite. Keycloak, Auth0, Okta, Google, and other compliant providers also work.

OIDC supports three application styles, which differ along two axes — how tokens travel (cookies vs. `Authorization` headers) and how the framework responds when a request is not authenticated (an HTML redirect vs. a status code):

| Style                 | Factory         | Token transport         | When unauthenticated     | Session endpoints |
|-----------------------|-----------------|-------------------------|--------------------------|-------------------|
| Server-rendered (SSR) | `OIDC.ssr(...)` | Cookies                 | Redirect to login (HTML) | Yes               |
| Single-page app (SPA) | `OIDC.spa(...)` | Cookies                 | `401` / `403` / `503`    | Yes               |
| API                   | `OIDC.api(...)` | `Authorization: Bearer` | `401` / `403` / `503`    | No                |

You configure the provider once with `OIDCConfig`, create an `OIDC` instance for your application style, install the session endpoints (for browser styles), then attach `oidc.authenticated()` (or a role check) to anything you want protected.

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

The framework discovers the authorize, token, userinfo, jwks, and (optional) logout endpoints from the issuer's `/.well-known/openid-configuration` document. To bypass discovery, set the endpoints explicitly instead of `issuer` (you must provide at least authorize, token, userinfo, and jwks):

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

`OIDCConfig` is validated when you `build()` it. The key rules:

- `clientId` is always required.
- `clientSecret` is required for confidential clients. Set `publicClient(true)` for CLI, native, and/or desktop clients that authenticate with PKCE and no secret.
- Either `issuer` or all four core endpoints (authorize, token, userinfo, jwks) must be provided.
- `scopes` must include `openid`. The default scopes are `openid`, `profile`, `email`, and `offline_access` (so refresh tokens are issued).
- All endpoints must be secure (HTTPS) URIs, except when using `localhost`.

### Token validation

By default (`validateAccessToken(true)`) access tokens are validated locally as JWTs against the provider's JWKS. To validate via [RFC 7662 token introspection](https://www.rfc-editor.org/rfc/rfc7662) instead — which also supports opaque (non-JWT) access tokens — set `validateAccessToken(false)` and provide an `introspectionEndpoint`:

~~~~ java
var config = OIDCConfig.builder()
    .issuer("https://auth.example.com")
    .clientId("my-app")
    .clientSecret(secret)
    .validateAccessToken(false)
    .introspectionEndpoint(URI.create("https://auth.example.com/oauth2/introspect"))
    .build();
~~~~

> **Note:** Introspection requires client authentication, so `publicClient(true)` is incompatible with `validateAccessToken(false)`. Some providers (FusionAuth included) do not advertise the introspection endpoint in discovery, so set it explicitly.

## Server-rendered apps (SSR)

`OIDC.ssr(config)` returns an `OIDC<JWT>` for a cookie-based, redirect-driven app. Install the **session endpoints** once so the OIDC paths (login, callback, logout) are handled, then protect any subset of routes:

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

  var oidc = OIDC.ssr(config);

  try (var web = new Web()) {
    web.install(SecurityHeaders.defaults());
    web.install(new OriginChecks());
    web.install(OIDC.sessionEndpoints(config));

    web.get("/", (req, res) -> res.getWriter().write("public landing"));

    web.prefix("/app", app -> {
      app.install(oidc.authenticated());
      app.get("/me", (req, res) -> {
        JWT jwt = OIDC.jwt();
        res.getWriter().write("Hi " + jwt.subject());
      });
    });

    web.start(8080);
  }
}
~~~~

`OIDC.sessionEndpoints(config)` is a separate middleware that owns the login/callback/logout paths; the `oidc` instance only carries protection and the user accessors. Install the session endpoints once per browser client.

The entire login and logout flow then becomes two links in your UI:

~~~~ html
<a href="/login">Sign in</a>
<a href="/logout">Sign out</a>
~~~~

Clicking **Sign in** redirects the browser through the provider, lands on `/oidc/return`, sets the token cookies, and redirects to `postLoginPage`. Clicking **Sign out** clears the cookies, calls the provider's end-session endpoint, and redirects to `postLogoutPage`.

### Browser settings

The cookie names, OIDC paths, and post-login/logout destinations are configured with `BrowserSettings`, not `OIDCConfig`. Pass it to both the `OIDC` instance and the session endpoints when you want to override the defaults:

~~~~ java
var browser = BrowserSettings.builder()
    .loginPath("/login")
    .postLoginPage("/dashboard")
    .postLogoutPage("/goodbye")
    .build();

var oidc = OIDC.ssr(config, browser, jwt -> jwt);   // translator required with this overload
web.install(OIDC.sessionEndpoints(config, browser));
~~~~

`BrowserSettings` defaults:

| Setting              | Default               | Purpose                              |
|----------------------|-----------------------|--------------------------------------|
| `loginPath`          | `/login`              | Initiates the OIDC flow              |
| `callbackPath`       | `/oidc/return`        | Authorization-code callback          |
| `logoutPath`         | `/logout`             | Initiates RP-initiated logout        |
| `logoutReturnPath`   | `/oidc/logout-return` | Post-logout redirect target          |
| `postLoginPage`      | `/`                   | Where to land after login            |
| `postLogoutPage`     | `/`                   | Where to land after logout           |
| `errorPage`          | `/`                   | Where to land on an auth error       |
| `stateCookieName`    | `oidc_state`          | CSRF state cookie                    |
| `returnToCookieName` | `oidc_return_to`      | Remembers the post-login destination |

Tokens default to `access_token`, `refresh_token`, and `id_token` cookies (30-day refresh max age). You can also supply a custom `tokenReader`/`tokenWriter` for non-cookie transports.

## Single-page apps (SPA)

`OIDC.spa(config)` is the same cookie-based flow as SSR, but unauthenticated requests get a status code (`401`/`403`/`503`) instead of an HTML redirect — appropriate for a JavaScript front end that calls JSON endpoints. It also uses the session endpoints:

~~~~ java
var oidc = OIDC.spa(config);
web.install(OIDC.sessionEndpoints(config));
web.prefix("/api", api -> {
  api.install(oidc.authenticated());
  api.get("/profile", (req, res) -> writeJson(res, OIDC.jwt()));
});
~~~~

> **NOTE**: It is highly recommended that SPAs do not use OIDC as public clients. `web` handles all the heavy lifting of securely managing OIDC, PCKE, cookies, and everything else. Building a SPA that must be a public client is outside of the scope of this documentation. 

## APIs

`OIDC.api(config)` is for services authenticated by a bearer token rather than cookies. There are no session endpoints — clients obtain tokens elsewhere (an SSR/SPA front end, a `client_credentials` grant, etc.) and send them in the `Authorization: Bearer` header. Unauthenticated requests get status codes.

~~~~ java
var oidc = OIDC.api(config);
web.prefix("/api/v1", api -> {
  api.install(oidc.authenticated());
  api.get("/things", listThings);
});
~~~~

The access and refresh transport for APIs is configured with `APISettings` (pass it to `OIDC.api(config, apiSettings, translator)`). By default the access token is read from `Authorization: Bearer`, the refresh token from `X-Refresh-Token`, and refreshed tokens are written back to the `X-Access-Token` / `X-Refresh-Token` response headers. The header names are available as constants: `APISettings.AUTHORIZATION`, `APISettings.X_ACCESS_TOKEN`, `APISettings.X_REFRESH_TOKEN`.

API authentication supports the `client_credentials` grant. Because it validates the access token, that token must be a JWT unless you enable introspection (`validateAccessToken(false)`), which also accepts opaque tokens.

## Accessing the user

Inside a protected handler, the static `OIDC.jwt()` returns the bound JWT, or throws `UnauthenticatedException` (which renders as `401`) if no JWT is present:

~~~~ java
app.get("/me", (req, res) -> {
  JWT jwt = OIDC.jwt();
  String email = jwt.getString("email");
  res.getWriter().write("hello " + email);
});
~~~~

`OIDC.optionalJWT()` returns an `Optional<JWT>` if you'd rather not throw:

~~~~ java
web.get("/maybe", (req, res) -> {
  String greeting = OIDC.optionalJWT()
      .map(jwt -> "hello " + jwt.subject())
      .orElse("hello stranger");
  res.getWriter().write(greeting);
});
~~~~

### Translating the JWT to a domain object

Each factory has an overload that takes a `Function<JWT, U>` translator, so `oidc.user()` returns your own user type. The translator is applied lazily on every call.

~~~~ java
record AppUser(String id, String email, Set<String> roles) {}

Function<JWT, AppUser> translator = jwt -> new AppUser(
    jwt.subject(),
    jwt.getString("email"),
    new HashSet<>(jwt.getList("roles", String.class))
);

OIDC<AppUser> oidc = OIDC.ssr(config, translator);

// Inside a handler:
AppUser user = oidc.user();              // throws UnauthenticatedException if no JWT
Optional<AppUser> maybe = oidc.optionalUser();
~~~~

## Role-based access

If your JWT carries a `roles` claim (the default `roleExtractor` reads `jwt.getList("roles", String.class)`), you can require roles directly. These are instance methods on the `OIDC` object, so they apply your chosen application style:

~~~~ java
web.prefix("/admin", admin -> {
  admin.install(oidc.authenticated());
  admin.install(oidc.hasAnyRole("admin", "superuser"));
  admin.get("/users", listAllUsers);
  admin.delete("/users/{id}", deleteUser, oidc.hasAllRoles("admin"));
});
~~~~

- `authenticated()` requires a valid identity.
- `hasAnyRole(String...)` requires **at least one** of the listed roles.
- `hasAllRoles(String...)` requires **every** listed role.
- `authorized(Authorizer)` runs a custom predicate against the request and JWT.

When the role check fails the response is a `403` (rendered as a redirect or status code per the application style); a missing identity is a `401`. The role middlewares depend on `authenticated()` having run earlier in the chain.

If your provider exposes roles under a different claim, supply a custom extractor on the config:

~~~~ java
var config = OIDCConfig.builder()
    .issuer("https://auth.example.com")
    .clientId("my-app")
    .clientSecret(secret)
    .roleExtractor(jwt -> Set.copyOf(jwt.getList("groups", String.class)))
    .build();
~~~~

## Token refresh

Refresh is transparent. When `authenticated()` finds an invalid access token but a usable refresh token, it refreshes the tokens, writes the new ones using the configured token writer (cookies for SSR/SPA, response headers for APIs), and binds the new JWT to the request. If the identity provider can't be reached, the request fails with a `503` (`ServiceUnavailableException`).

## Putting it together

See the [Sample Application](../sample-application/) for a complete SSR server that uses OIDC, role checks, JSON bodies, static files, and security headers together. For testing OIDC-protected routes, the [`OIDCTestFixture`](../testing/#oidctestfixture) drives the full login flow against a real provider.
