---
layout: docs
title: Message Bundles
description: Looking up localized display text from properties files laid out to mirror the URL space.
weight: 77
---

## Overview

`Messages` (in `org.lattejava.web`) looks up display text for the current request from `.properties` files under `web/messages` in the project's base directory. The files are arranged in directories that match the path of the request. This allows the templates and message bundles to mirror each other on disk.

Message bundles can also be localized in the same manner that `ResourceBundle`s are localized. The lookup chain is the same using the different parts of the `Locale`. See the information below for the lookup chain.

The `Messages` object takes the request in the constructor and can be created any number of times since it doesn't store any state. Don't need to worry about loading or caching of the `.properties` files, the `Message` object handles all of that. 

~~~~ java
import module org.lattejava.web;

web.get("/admin/users/edit", (req, res) -> {
  Messages messages = new Messages(req);
  String title = messages.get("title");
  String saved = messages.get("saved", user.name());
  // ...
});
~~~~

`Messages` can also be used in templates and any other location that the `HTTPRequest` is available. `JTETemplates` always has access to the request, so it can instantiate a `Messages` object as well:

~~~~ html
@import org.lattejava.http.server.HTTPRequest
@import org.lattejava.web.Messages
@param HTTPRequest request

<h1>${new Messages(request).get("title")}</h1>
~~~~

## Files

| Item     | Value                                                                                                           |
|----------|-----------------------------------------------------------------------------------------------------------------|
| Location | `<baseDir>/web/messages`, where `baseDir` is set with `Web.baseDir(Path)` and defaults to the working directory |
| Format   | `java.util.Properties` text format                                                                              |
| Encoding | UTF-8                                                                                                           |
| Naming   | `<base>.properties`, `<base>_<lang>.properties`, `<base>_<lang>_<COUNTRY>.properties`                           |

The naming follows the `ResourceBundle` convention. A small site might look like this:

~~~~
web/messages/
├── index.properties
├── index_de.properties
├── users.properties
└── admin/
    ├── index.properties
    └── users/
        ├── index.properties
        ├── edit.properties
        └── edit_de.properties
~~~~

Files are read on first use and cached. Each time a `Messages` object is created, the file system is checked for any updates. Any new or modified files are reloaded. A path that resolves outside the messages directory is treated as missing, and a directory that does not exist at all simply defines no keys.

## Lookup

Two axes combine to decide which file answers a lookup: the request path first, then the locale.

### Path hierarchy

A request path names a leaf file (i.e. `edit.properties`), and every directory above it adds a check for a file named `index.properties`. A path ending in a slash maps to the `index.properties` file directly. This is the same mapping [templates](../templates/#template-name-resolution) use, so `/users` and `/users/` are different pages here as well.

| Request path        | File names, most specific first                                                                             |
|---------------------|-------------------------------------------------------------------------------------------------------------|
| `/`                 | `index.properties`                                                                                          |
| `/users`            | `users.properties`, `index.properties`                                                                      |
| `/users/`           | `users/index.properties`, `index.properties`                                                                |
| `/admin/users/edit` | `admin/users/edit.properties`, `admin/users/index.properties`, `admin/index.properties`, `index.properties` |

The first file that defines a key wins. A key set in `admin/index.properties` is shared by every page under `/admin/`, and a page can override it in its own file.

### Locale chain

Each base name expands into the locale candidates for the request, most specific first. For a `de_DE` request, `admin/users/edit` is tried as:

1. `admin/users/edit_de_DE.properties`
2. `admin/users/edit_de.properties`
3. `admin/users/edit.properties`

The locale comes from the request's `Accept-Language` header (`HTTPRequest.getLocale()`, the highest-weighted value, or the JVM default when the header is absent). Only the preferred locale is used; lower-weighted `Accept-Language` entries are not tried. To pin the locale, pass it to the constructor:

~~~~ java
Messages messages = new Messages(req, Locale.GERMAN);
~~~~

### Combined order

Every locale variant of one base name is consulted before moving up to the parent directory:

~~~~
admin/users/edit_de_DE  →  admin/users/edit_de  →  admin/users/edit
admin/users/index_de_DE →  admin/users/index_de →  admin/users/index
admin/index_de_DE       →  admin/index_de       →  admin/index
index_de_DE             →  index_de             →  index
~~~~

So for `/admin/users/edit` in `de_DE`, a key in `admin/users/edit.properties` beats the same key in `admin/index_de.properties`. A message written for the page wins over a translation written for the site.

## Reading messages

| Method               | Result                                                             |
|----------------------|--------------------------------------------------------------------|
| `get(key)`           | The value, or `MissingMessageException` if no file defines the key |
| `get(key, args...)`  | The value formatted with `MessageFormat` in the request's locale   |
| `find(key)`          | The value, or `null` if no file defines the key                    |
| `find(key, args...)` | The formatted value, or `null`                                     |
| `has(key)`           | `true` if a file in the chain defines the key                      |
| `locale()`           | The locale lookups are performed for                               |

### Formatting

When arguments are supplied, the value is a `MessageFormat` pattern evaluated in the request's locale, so numbers and dates are formatted per locale:

~~~~
greeting=Hello {0}
count=Total {0,number}
~~~~

~~~~ java
messages.get("greeting", user.name());   // Hello Brian
messages.get("count", 1234.5);           // Total 1,234.5 (en) or Total 1.234,5 (de)
~~~~

When no arguments are supplied, the value is returned verbatim. Plain text can contain apostrophes and braces without `MessageFormat` escaping:

~~~~
apostrophe=Don't panic
braces=Use {curly} braces
~~~~

An empty value (`key=`) counts as defined and returns `""`.

### Missing keys

`get` throws `MissingMessageException` for an undefined key. It extends `RuntimeException` rather than `HTTPException`, so the default [error renderer](../exception-handling/) returns a plain 500 instead of writing the key and path to the client.

Use `find` or `has` when a missing key is expected:

~~~~ java
String hint = messages.find("hint");
if (hint != null) {
  // ...
}
~~~~

## Example

Given these files:

~~~~
# web/messages/index.properties
title=Home
greeting=Hello {0}

# web/messages/index_de.properties
title=Startseite
greeting=Hallo {0}

# web/messages/admin/index.properties
title=Admin

# web/messages/admin/users/edit.properties
title=Edit User
save=Save

# web/messages/admin/users/edit_de.properties
save=Speichern
~~~~

Lookups resolve as follows:

| Request                        | `Accept-Language` | `get("title")` | `get("save")`       |
|--------------------------------|-------------------|----------------|---------------------|
| `/`                            | `en`              | `Home`         | throws              |
| `/`                            | `de`              | `Startseite`   | throws              |
| `/admin`                       | `en`              | `Home`         | throws              |
| `/admin/`                      | `en`              | `Admin`        | throws              |
| `/admin/users/edit`            | `en`              | `Edit User`    | `Save`              |
| `/admin/users/edit`            | `de`              | `Edit User`    | `Speichern`         |

Note that `/admin` (no trailing slash) looks for `admin.properties`, which does not exist, and falls through to `index.properties`. `/admin/` starts at `admin/index.properties`.

## Testing

You can access `Messages` in tests by constructing an instance with the path (and optionally the `Locale`) you are testing:

~~~~ java
try (var web = new Web()) {
  web.get("/admin/users/edit", (req, res) -> {
       res.getWriter().write(new Messages(req).get("save"));
     })
     .start(PORT);

  Messages messages = new Messages("/admin/users/edit");
  new WebTest(PORT)
      .withHeader("Accept-Language", "de")
      .get("/admin/users/edit")
      .assertBodyAs(new StringBodyAsserter(), s -> s.equalTo(messages.get("save"));
}
~~~~

The `Messages` class has a number of constructors that allow you to create an instance for testing. Refer to the Javadoc for the different options.