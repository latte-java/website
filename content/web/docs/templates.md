---
layout: docs
title: Templates
description: Rendering JTE templates to the HTTP response or to a String.
weight: 75
---

## Overview

`JTETemplates` (in `org.lattejava.web.jte`) renders [JTE](https://jte.gg/) templates. The `web` library depends on `gg.jte:jte`, so JTE is available to any project that uses Latte Web. Projects created with `latte init web` include a starter `web/templates/index.jte`.

There are two ways to use it:

- The `html(...)` methods write the rendered HTML straight to the `HTTPResponse`. They set the content type to `text/html; charset=utf-8` and leave the status alone (so if something upstream set a status, it is preserved; otherwise it defaults to 200).
- The `render(...)` methods return the rendered output as a `String`. Use these for non-HTTP scenarios such as composing email bodies or building reusable fragments.

In every template the `request` and `response` objects are available as the `request` and `response` parameters.

## Constructing

The default constructor reads `.jte` sources from `web/templates` (relative to the working directory), compiles classes to `build/jte-classes`, and renders as HTML:

~~~~ java
import module org.lattejava.web;

var templates = new JTETemplates();
~~~~

You can point it at a different template directory, and optionally a different compiled-classes directory:

~~~~ java
var templates = new JTETemplates(Paths.get("src/templates"));
var templates = new JTETemplates(Paths.get("src/templates"), Paths.get("target/jte-classes"));
~~~~

For deployed applications, supply your own `TemplateEngine` (for example a precompiled engine) and pass it in:

~~~~ java
import module gg.jte;

TemplateEngine engine = TemplateEngine.createPrecompiled(ContentType.Html);
var templates = new JTETemplates(engine);
~~~~

## Template name resolution

The path-based `html(...)` overloads derive the template name from the request path:

| Request path | Template       |
|--------------|----------------|
| `/`          | `index.jte`    |
| `/foo`       | `foo.jte`      |
| `/foo/`      | `foo/index.jte`|
| `/foo/bar`   | `foo/bar.jte`  |

You can also name the template explicitly with the overloads that take a `name` argument (for example `"foo.jte"`).

## Model binding

- A single non-`Map` model is bound to a template parameter named `model`.
- A `Map<String, Object>` binds each entry to a template parameter named by the map key.

## Rendering a page

A handler that renders the template for the current request path, passing a model object:

~~~~ java
import module org.lattejava.web;

void main() {
  var templates = new JTETemplates();

  new Web()
      .get("/", (req, res) -> templates.html(req, res, new HomePage("Latte")))
      .start(8080);
}

record HomePage(String title) {}
~~~~

The matching `web/templates/index.jte`, which receives the model as the `model` parameter:

~~~~ html
@import HomePage
@param HomePage model

<!DOCTYPE html>
<html>
<head><title>${model.title()}</title></head>
<body>
  <h1>Welcome to ${model.title()}!</h1>
</body>
</html>
~~~~

The starter `web/templates/index.jte` created by `latte init web` is minimal:

~~~~ html
Welcome to Latte Java!
~~~~

### Binding multiple parameters

Pass a `Map` to bind several named parameters at once:

~~~~ java
web.get("/dashboard", (req, res) ->
    templates.html(req, res, Map.of(
        "user", currentUser,
        "stats", stats)));
~~~~

~~~~ html
@import com.example.User
@import com.example.Stats
@param User user
@param Stats stats

<p>Hello ${user.name()}, you have ${stats.unread()} unread messages.</p>
~~~~

### Rendering a named template

When the template name doesn't follow the request path, name it explicitly:

~~~~ java
web.get("/profile/{id}", (req, res) ->
    templates.html("profile.jte", req, res, loadUser(req)));
~~~~

## Rendering to a String

The `render(...)` methods return the output instead of writing it to a response. This is handy for email bodies or fragments:

~~~~ java
String body = templates.render("email/welcome.jte", new WelcomeEmail(user));
mailer.send(user.email(), "Welcome!", body);
~~~~

`render` has overloads for no parameters, a single `model`, and a `Map<String, Object>`:

~~~~ java
String a = templates.render("banner.jte");
String b = templates.render("banner.jte", model);
String c = templates.render("banner.jte", Map.of("title", "Hello"));
~~~~

Note that `render` does not touch an `HTTPResponse`, so the `request` and `response` parameters are not available in templates rendered this way.
