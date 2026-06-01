---
layout: docs
title: Publishing
description: Publish your project's artifacts to the Latte public repository using the latte publish process.
weight: 29
---

## Overview

The Latte public repository accepts artifacts through an authenticated publish API. Rather than handing build tools long-lived S3/R2 credentials, Latte uses the OAuth tokens from [`latte login`](../authentication/) when calling the API.

You enable this by adding the `latte()` process to your project's `publishWorkflow`. Projects created with `latte init` already include it.

## Prerequisite: log in

Before publishing, you must first login:

~~~~ shell
$ latte login
~~~~

If no access token is stored when you publish, the build fails with:

~~~~
You are not logged in to the Latte repository. Run [latte login] before publishing.
~~~~

See [Authentication](../authentication/) for details.

## The `latte()` publish process

Add `latte()` to your `publishWorkflow` block:

~~~~ groovy
publishWorkflow {
  latte()
}
~~~~

`latte()` is a **publish-only** process — it does not fetch artifacts. Fetching from the public repository is handled by the `url` or `s3` processes. See [Workflows](../workflows/) for the full list of processes.

## How it works

For each publication, the `latte()` process:

1. Loads your stored access and refresh tokens.
2. POSTs to `https://api.lattejava.org/api/v1/publish/{group}` with the tokens to request a presigned URL for the artifact.
3. The API responds with a presigned URL.
4. If the API returns refreshed tokens (via the `X-Refresh-Token` response header), Latte writes them back to your configuration file so the next publish uses the new tokens.
5. Uploads the artifact bytes to the presigned URL.

On success, you'll see messages like this for the artifacts being published:

~~~~
Published [my-lib-0.1.0.jar] to the Latte repository [org/example/my-lib/0.1.0/my-lib-0.1.0.jar]
~~~~

## Authorization

To publish to a group you must be an active owner or contributor of a **verified** group. You'll receive an error message if any part of the process failed.

## Releasing

When you run a [release](../releasing/), the `release` target publishes through this same `publishWorkflow`, so a successful `latte login` is a prerequisite for releasing to the Latte public repository.
