---
title: "Documentation"
layout: docs
description: "Latte web framework documentation"
weight: 1
---

Welcome to the Latte `web` documentation. The pages below walk through everything you need to build a web application — from a minimal "Hello, world!" to a fully authenticated JSON API.

## Getting Started

- [Getting Started](getting-started/) - Install Latte, scaffold a project, and run the server
- [Project Layout](project-layout/) - What `latte init web` creates and how it's wired together

## Building Blocks

- [Routing](routing/) - Verbs, path parameters, prefixes, and how 404/405 are handled
- [Handlers](handlers/) - Writing request handlers and working with `HTTPRequest` and `HTTPResponse`
- [Request Bodies](request-bodies/) - `BodyHandler`, `BodySupplier`, and the built-in JSON supplier
- [Middleware](middleware/) - The pipeline, ordering, and writing your own

## Production Features

- [Static Files](static-files/) - Serving assets with caching and traversal protection
- [Security Headers](security-headers/) - The `SecurityHeaders` middleware and what each header does
- [CSRF Protection](csrf-protection/) - The `OriginChecks` middleware and `SameSite` cookie strategy
- [Exception Handling](exception-handling/) - Mapping exceptions to HTTP status codes
- [OIDC Authentication](oidc/) - Login, callback, refresh, logout, and role-based access

## End-to-End

- [Sample Application](sample-application/) - A complete server pulling all of the above together
