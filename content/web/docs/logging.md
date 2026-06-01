---
layout: docs
title: Logging
description: Configuring the swappable LoggerFactory and log level used by the web and HTTP layers.
weight: 65
---

## Overview

The web and underlying HTTP layers log through a swappable `LoggerFactory`. When you don't configure one, `Web` uses `WebPrintStreamLoggerFactory.FACTORY` by default. That factory always returns a single shared `WebPrintStreamLogger` that writes to `System.out`, prefixing each line with an ISO-8601 offset date-time formatted in the system default time zone (e.g. `2026-04-27T13:45:23.689-04:00`).

Once the server is up, `Web` logs a single line announcing where the application is reachable:

~~~~
Web application is available at [http://localhost:8080]
~~~~

The URL is built from the bind address and port, so it points at `localhost` when bound to any-local, or the actual host/port otherwise.

## Setting the log level

`Web.logLevel(Level)` sets the level applied to the logger that `Web` retrieves from its `LoggerFactory`. The `Level` type lives in `org.lattejava.http.log` and has four values: `Trace`, `Debug`, `Info`, and `Error`.

~~~~ java
import module org.lattejava.http;
import module org.lattejava.web;

void main() {
  new Web()
      .logLevel(Level.Debug)
      .get("/", ctx -> ctx.write("Hello"))
      .start(8080);
}
~~~~

`logLevel` must be called **before** `start(int)`. Calling it after the server has started throws `IllegalStateException`.

## Swapping the LoggerFactory

`Web.loggerFactory(LoggerFactory)` replaces the factory used by this `Web` instance and the underlying HTTP server. Provide your own implementation to route web and HTTP logs into your application's logging system:

~~~~ java
import module org.lattejava.http;
import module org.lattejava.web;

void main() {
  LoggerFactory factory = klass -> myLoggerAdapterFor(klass);

  new Web()
      .loggerFactory(factory)
      .get("/", ctx -> ctx.write("Hello"))
      .start(8080);
}
~~~~

Like `logLevel`, `loggerFactory` must be called **before** `start(int)` and throws `IllegalStateException` if called afterward.

## Capturing logs in tests

`WebPrintStreamLoggerFactory` has a `WebPrintStreamLoggerFactory(PrintStream)` constructor that lets you point the logger at any stream instead of `System.out`. Wrap a `ByteArrayOutputStream` to capture log output in a test without redirecting `System.out`:

~~~~ java
import module java.base;
import module org.lattejava.http;
import module org.lattejava.web;
import org.lattejava.web.log.WebPrintStreamLoggerFactory;

void main() throws Exception {
  var captured = new ByteArrayOutputStream();
  var factory = new WebPrintStreamLoggerFactory(new PrintStream(captured, true));

  var web = new Web()
      .loggerFactory(factory)
      .logLevel(Level.Debug)
      .get("/", ctx -> ctx.write("Hello"))
      .start(8080);

  // ... exercise the server, then assert against captured.toString()

  web.close();
}
~~~~
