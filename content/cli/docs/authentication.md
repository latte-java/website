---
layout: docs
title: Authentication
description: Log in to the Latte identity provider so you can publish artifacts to the Latte public repository.
weight: 14
---

## Overview

Publishing artifacts to the Latte public repository requires authentication. The `latte login` command authenticates you with the Latte identity provider and stores the resulting OAuth tokens locally. The [`latte()` publish process](../publishing/) then reuses those tokens to publish on your behalf — you never have to manage repository credentials directly.

## Logging in

~~~~ shell
$ latte login
~~~~

This command opens your system browser to the Latte login page. If you don't already have a Latte account, you can register for a new account from here.

Once you have registered or logged in, you should see a message like this:

~~~~
Logged in as [you@example.com]
~~~~

## Where credentials are stored

Tokens are written to the global configuration file at `~/.config/latte/config.properties` under these keys:

| Key                       | Description                                                              |
|---------------------------|--------------------------------------------------------------------------|
| `latte.auth.accessToken`  | The OAuth access token, used to authorize publish requests.              |
| `latte.auth.refreshToken` | The OAuth refresh token, used to obtain new access tokens automatically. |

The file is written atomically, and on POSIX systems its permissions are restricted to owner read/write (`rw-------`). Any other properties already in the file are preserved. The access token is refreshed automatically during publishing; rotated tokens are written back to this same file.

This is the same file that holds your other [global configuration variables](../variables/), so logging in does not disturb your other settings.

## Logging out

~~~~ shell
$ latte logout
~~~~

This removes the `latte.auth.accessToken` and `latte.auth.refreshToken` keys from `~/.config/latte/config.properties`, leaving all other properties intact. You can also edit the configuration file manually and delete these keys to effectively log out.

## Next steps

- [Publishing](../publishing/) - Publish your artifacts to the Latte repository using the tokens from `latte login`.
- [Releasing](../releasing/) - Cut a release, which publishes through the same mechanism.
