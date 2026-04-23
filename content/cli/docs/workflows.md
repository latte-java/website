---
layout: docs
title: Workflows
description: Workflows define the ordered list of processes Latte runs to fetch dependencies and publish artifacts.
weight: 32
---

## Overview

A workflow is an ordered list of processes that Latte runs to resolve dependencies or publish artifacts. When Latte needs to fetch an artifact, it tries each process in order and uses the first one that produces a result. When Latte publishes an artifact, it runs every process in order so the artifact ends up in every configured location.

Every project file defines two workflows:

* **`workflow { ... }`** — the fetch-time workflow. Runs whenever Latte needs to download a dependency.
* **`publishWorkflow { ... }`** — the publish-time workflow. Runs whenever Latte publishes a built artifact (for example, during a release).

The two blocks appear at the top level of the `project(...) { }` definition:

~~~~ groovy
project(group: "com.example", name: "my-app", version: "1.0.0", licenses: ["MIT"]) {
  workflow {
    standard()
  }
  publishWorkflow {
    cache()
  }

  dependencies { /* ... */ }
}
~~~~

> **Ordering matters.** Workflows must be declared *before* `dependencies { }` in the project file — Latte parses the `project(...)` block linearly and needs the workflows in place before it can resolve dependency artifacts.

For concrete repository setup examples (public, private, Maven Central, S3), see [Repositories](../repositories/). This page documents the DSL itself.

## Fetch vs. publish semantics

| Workflow | Trigger | Behavior |
|----------|---------|----------|
| `workflow` | Resolving or downloading a dependency | Tries processes in order, stops at the first success. |
| `publishWorkflow` | Publishing a built artifact | Runs every process in order; artifact lands in all configured locations. |

## The `workflow` block

Inside `workflow { }` you can use any of the following methods:

| Method | Purpose |
|--------|---------|
| `standard()` | Shorthand for a sensible default fetch + publish setup. |
| `fetch { ... }` | Explicitly declare the ordered list of fetch processes. |
| `publish { ... }` | Explicitly declare the ordered list of publish processes (for the implicit publish side of `workflow`). |
| `semanticVersions { ... }` | Provide version mappings for artifacts whose Maven versions aren't SemVer-compliant. |

### `standard()`

Expands to the following defaults:

~~~~ groovy
fetch {
  cache()
  url(url: "https://repository.lattejava.org")
  maven(url: "https://repo1.maven.org/maven2")
}
publish {
  cache()
}
~~~~

Use `standard()` when you want the default behavior (check the local cache, then the Latte public repository, then Maven Central) and you don't need custom repositories.

`standard()` only populates the implicit publish side of `workflow`. It does **not** add anything to `publishWorkflow { }`.

### `fetch { ... }` and `publish { ... }`

Use these when you need to customize the list of fetch or publish processes. Both accept the same set of [process methods](#processes).

~~~~ groovy
workflow {
  fetch {
    cache()
    url(url: "https://repository.lattejava.org")
    maven(url: "https://repo1.maven.org/maven2")
  }
  publish {
    cache()
  }
}
~~~~

You can combine `standard()` with custom `fetch { }` or `publish { }` blocks — the processes from each are appended in the order they appear.

### `semanticVersions { ... }`

Maven does not enforce SemVer, but Latte does. When a transitive dependency has a non-SemVer version (for example, `1.0.0.Final`), declare a mapping so Latte can resolve it:

~~~~ groovy
workflow {
  standard()
  semanticVersions {
    mapping(id: "org.badver:badver:1.0.0.Final", version: "1.0.0")
  }
}
~~~~

See [Repositories — SemVer](../repositories/#semver) for the broader context on why Latte is strict about versions.

## The `publishWorkflow` block

`publishWorkflow { }` takes processes directly — it does not have an inner `fetch { }` or `publish { }`. Every process you declare runs, in order, every time Latte publishes an artifact.

~~~~ groovy
publishWorkflow {
  cache()
  s3(endpoint: "https://account-id.r2.cloudflarestorage.com", bucket: "my-repo",
     accessKeyId: global.repoAccessKey, secretAccessKey: global.repoSecretKey)
}
~~~~

In this example, publishing an artifact writes it into the local cache first, then uploads it to the S3 bucket.

If your project never publishes artifacts, you can omit `publishWorkflow { }` entirely or leave it empty.

## Processes

The following processes are available inside `fetch { }`, `publish { }`, and `publishWorkflow { }`. They are listed alphabetically.

### `cache`

Adds a cache process that reads from (during fetch) or writes to (during publish) the Latte local cache and the local Maven cache (`~/.m2/repository` by default).

~~~~ groovy
cache()
cache(dir: "/custom/latte/cache", mavenDir: "/custom/maven/cache", integrationDir: "/custom/integration/cache")
~~~~

| Attribute | Required | Default | Description |
|-----------|----------|---------|-------------|
| `dir` | No | Latte's cache directory | Override the Latte cache directory. |
| `mavenDir` | No | `~/.m2/repository` | Override the Maven cache directory. |
| `integrationDir` | No | Latte's cache directory | Override the integration-build cache directory. |

### `mavenCache`

A variant of `cache` that reads only from the Maven cache (`~/.m2/repository`), not the Latte cache. Useful when you want to interoperate with artifacts produced by Maven locally but do not want them to leak into the Latte side of your workflow.

~~~~ groovy
mavenCache()
mavenCache(dir: "/custom/maven/cache")
~~~~

| Attribute | Required | Default | Description |
|-----------|----------|---------|-------------|
| `dir` | No | `~/.m2/repository` | Override the Maven cache directory. |
| `integrationDir` | No | Latte's cache directory | Override the integration-build cache directory. |

### `url`

Adds an HTTP-based repository process. Supports HTTP Basic authentication.

~~~~ groovy
url(url: "https://repository.lattejava.org")
url(url: "https://my-repo.internal.example.com", username: global.repoUsername, password: global.repoPassword)
~~~~

| Attribute | Required | Default | Description |
|-----------|----------|---------|-------------|
| `url` | **Yes** | — | Base URL of the HTTP repository. |
| `username` | No | — | HTTP Basic username. |
| `password` | No | — | HTTP Basic password. |

### `maven`

Adds a Maven-style HTTP repository process. When Latte fetches through this process, it reads the Maven POM and translates dependencies into the Latte dependency model. See [Repositories — Maven Central](../repositories/#maven-central) for the translation rules.

~~~~ groovy
maven()
maven(url: "https://repo1.maven.org/maven2")
maven(url: "https://maven.example.com/releases", username: "user", password: "pass")
~~~~

| Attribute | Required | Default | Description |
|-----------|----------|---------|-------------|
| `url` | No | `https://repo1.maven.org/maven2` | Base URL of the Maven repository. |
| `username` | No | — | HTTP Basic username. |
| `password` | No | — | HTTP Basic password. |

### `s3`

Adds an S3-compatible object storage process. Works with AWS S3, Cloudflare R2, MinIO, Backblaze B2, DigitalOcean Spaces, and any other S3-compatible service. See [Repositories — S3-compatible providers](../repositories/#s3-compatible-providers) for a catalogue.

~~~~ groovy
s3(endpoint: "https://account-id.r2.cloudflarestorage.com", bucket: "my-repo",
   accessKeyId: global.repoAccessKey, secretAccessKey: global.repoSecretKey)
~~~~

| Attribute | Required | Default | Description |
|-----------|----------|---------|-------------|
| `endpoint` | **Yes** | — | S3-compatible endpoint URL. |
| `bucket` | **Yes** | — | Bucket name. |
| `accessKeyId` | **Yes** | — | Access key. |
| `secretAccessKey` | **Yes** | — | Secret key. |
| `region` | No | `auto` | Region. Use `auto` for Cloudflare R2; use the actual region for AWS S3. |

## Common patterns

### Default open-source project

~~~~ groovy
workflow {
  standard()
}
publishWorkflow {
  cache()
}
~~~~

### Public repository plus a private S3 bucket

~~~~ groovy
workflow {
  fetch {
    cache()
    url(url: "https://repository.lattejava.org")
    maven(url: "https://repo1.maven.org/maven2")
    s3(endpoint: "https://account-id.r2.cloudflarestorage.com", bucket: "my-repo",
       accessKeyId: global.repoAccessKey, secretAccessKey: global.repoSecretKey)
  }
  publish {
    cache()
  }
}
publishWorkflow {
  s3(endpoint: "https://account-id.r2.cloudflarestorage.com", bucket: "my-repo",
     accessKeyId: global.repoAccessKey, secretAccessKey: global.repoSecretKey)
}
~~~~

### Non-SemVer Maven dependencies

~~~~ groovy
workflow {
  standard()
  semanticVersions {
    mapping(id: "org.badver:badver:1.0.0.Final", version: "1.0.0")
  }
}
~~~~
