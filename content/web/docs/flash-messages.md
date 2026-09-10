---
layout: docs
title: Flash Messages
description: Queuing messages that survive a redirect and displaying them on the next page.
weight: 76
---

## Overview

A handler that processes a form usually redirects afterward (POST, redirect, GET). Any message it wants to show, such as "Settings saved", has to outlive the redirect and reach the page that is finally rendered. Flash messages solve this with two pieces:

- `Flash` (in `org.lattejava.web`) is a thin wrapper around the request. Handlers use it to add messages and templates use it to read them.
- `FlashMessages` (in `org.lattejava.web.middleware`) is the middleware that carries the messages between requests in a cookie named `flash`.

Install the middleware globally, then use `Flash` anywhere the request is available:

~~~~ java
import module org.lattejava.web;

void main() {
  var templates = new JTETemplates();

  try (var web = new Web()) {
    web.install(new FlashMessages())
       .get("/settings", (req, res) -> templates.html(req, res, Map.of()))
       .post("/settings", (req, res) -> {
         // ... save the settings ...
         new Flash(req).addMessage("Settings saved");
         res.sendRedirect("/settings");
       })
       .start(8080);
  }
}
~~~~

## Adding messages

`new Flash(req)` wraps the current request. `addMessage(String)` queues a message and returns the `Flash` for chaining:

~~~~ java
web.post("/users", (req, res) -> {
  // ... create the user ...
  new Flash(req)
      .addMessage("User created")
      .addMessage("A welcome email is on its way");
  res.sendRedirect("/users");
});
~~~~

The other methods are:

- `messages()` returns the queued messages as an unmodifiable list, in the order they were added. It is empty when there is no cookie or the cookie cannot be read.
- `hasMessages()` is a shortcut for `!messages().isEmpty()`.
- `clear()` discards every queued message, including any carried in from earlier requests.

`Flash` holds no state of its own. Reading parses the request's `flash` cookie, and adding or clearing replaces that cookie on the request. Every `new Flash(req)` created during a request therefore sees the same messages, whether it is created in a handler, a middleware, or a template.

## Displaying messages

`JTETemplates` always binds the `request` parameter, so a template reads the messages directly from it:

~~~~ html
@import org.lattejava.http.server.HTTPRequest
@import org.lattejava.web.Flash
@param HTTPRequest request

@for(String message : new Flash(request).messages())
  <div class="flash">${message}</div>
@endfor
~~~~

JTE escapes `${...}` output by default, which matters here: the cookie is neither signed nor encrypted, so the browser can edit it. Render messages with the same escaping you would apply to any other user-supplied text.

## Lifecycle

The middleware decides after each request what the browser's cookie should hold:

| Request outcome               | Effect on the flash cookie                                                  |
|-------------------------------|-----------------------------------------------------------------------------|
| `GET`, final status not `3xx` | Consumed. The cookie is cleared. Messages added during this GET are dropped |
| `GET`, final status `3xx`     | Carried forward                                                             |
| Any other method, any status  | Carried forward                                                             |
| Handler throws                | Left exactly as the browser sent it                                         |

"Carried forward" means the cookie is rewritten only if the handler added or removed messages. If the handler did not touch the messages, no `Set-Cookie` header is sent at all.

The rule of thumb: messages survive any number of redirects and non-GET requests, and are consumed by the first GET that renders a page. The page rendered by that GET is assumed to have displayed them, so a message is never shown twice.

~~~~
POST /settings          add "Settings saved", redirect     cookie written
GET  /settings          render the page                    cookie cleared
GET  /settings          render again                       no messages
~~~~

### Messages added during a GET

A message added during a GET that renders a page is visible to that request's template but is not carried to the next request. This is fine for a page that both computes and displays a notice. To show a message on a different page, add it and then redirect.

## Installing the middleware

`FlashMessages` has a no-argument constructor and is installed like any other [middleware](../middleware/):

~~~~ java
web.install(new FlashMessages());
~~~~

It can also be scoped to a prefix. Without the middleware, `Flash` still reads and updates the request's cookie during a request, but nothing reaches the browser, so messages never survive a redirect.

## The cookie

| Attribute  | Value                                                     |
|------------|-----------------------------------------------------------|
| Name       | `flash` (also `Flash.COOKIE_NAME`)                        |
| Value      | Base 64 encoded JSON (i.e. `{"messages":["...", "..."]}`) |
| `HttpOnly` | `true`                                                    |
| `SameSite` | `Lax`                                                     |
| `Path`     | `/`                                                       |
| `Secure`   | Auto-detected, as described in [Cookies](../cookies/)     |
| `Max-Age`  | Unset (browser session)                                   |

`SameSite=Lax` rather than the `Cookies` default of `Strict` lets the cookie ride along on the top-level GET that ends a redirect chain started at another site, such as returning from an OIDC provider after login.

A cookie value that cannot be decoded, is not the expected JSON shape, or contains anything other than strings in `messages` reads as no messages (`null` entries are dropped). Unreadable values are logged at `DEBUG` and cleared by the next consuming `GET` like any other cookie. Browsers cap a cookie at roughly 4 KB, so queue a few short messages, not a page of text.

## Testing

The `WebTest` client's [cookie jar](../testing/#the-cookie-jar) carries `Set-Cookie` values across requests, so a test can post, follow the redirect, and assert on the rendered messages:

~~~~ java
new WebTest(PORT)
    .post("/settings")
    .assertStatus(302)
    .reset(ResetItem.Request)
    .get("/settings")
    .assertStatus(200)
    .assertBodyAs(new StringBodyAsserter(), s -> s.contains("Settings saved"));
~~~~

## How it works

For those that are interested in how this all works, here's a summary.

Cookie headers have to be sent before the response body, and a handler that writes a body commits the response. In order to handle this situation, the `FlashMessages` middleware adds a `Set-Cookie` header before it forwards the request to the handler. This header sets the expiration of the flash cookie to `now`. This is how cookies are deleted from browsers. This cookie header is essentially "teed up" and will be sent back to the browser unless it is removed.

If the handler turns out to be another redirect, the `FlashMessages` middleware removes this header. This allows the browser to keep the current value of the cookie. This is done by checking if the handler set the status code in the response to a `3xx` value.

However, if the handler writes to the response (i.e. sends back HTML) or sets a non-`3xx` status code, then the `Set-Cookie` header to delete the cookie is sent back to the browser. The browser then deletes the cookie, thereby removing the flashed messages.