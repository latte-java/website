---
title: "http"
description: "A zero-dependency HTTP server for Java, powered by virtual threads"
layout: landing
github: "https://github.com/latte-java/http"
og_image: "/images/og/http.png"
---

## Install

Add to your Latte project:

```groovy
dependency(id: "org.lattejava:http:1.0.0")
```

Or install with the CLI:

```bash
latte install org.lattejava:http:1.0.0
```

## Quick Start

With Java 25 class-less main methods, getting a server running is just a few lines:

```java
import org.lattejava.http.server.HTTPListenerConfiguration;
import org.lattejava.http.server.HTTPServer;

void main() {
  var server = new HTTPServer().withHandler((req, res) -> {
                                  // Handler code goes here
                                })
                               .withListener(new HTTPListenerConfiguration(4242));
  server.start();
}
```

Here's the traditional class-based version:

```java
import org.lattejava.http.server.HTTPHandler;
import org.lattejava.http.server.HTTPListenerConfiguration;
import org.lattejava.http.server.HTTPServer;

public class Example {
  public static void main(String... args) {
    HTTPHandler handler = (req, res) -> {
      // Handler code goes here
    };

    HTTPServer server = new HTTPServer().withHandler(handler)
                                        .withListener(new HTTPListenerConfiguration(4242));
    server.start();
    // Use server
    server.close();
  }
}
```

Since `HTTPServer` implements `java.io.Closeable`, you can also use a try-with-resources block:

```java
import org.lattejava.http.server.HTTPHandler;
import org.lattejava.http.server.HTTPListenerConfiguration;
import org.lattejava.http.server.HTTPServer;

public class Example {
  public static void main(String... args) {
    HTTPHandler handler = (req, res) -> {
      // Handler code goes here
    };

    try (HTTPServer server = new HTTPServer().withHandler(handler)
                                             .withListener(new HTTPListenerConfiguration(4242))) {
      server.start();
      // When this block exits, the server will be shutdown
    }
  }
}
```

## Features

- **Virtual threads** - Built on Project Loom for simple, high-performance concurrency with blocking I/O
- **Zero dependencies** - Pure Java, nothing else
- **TLS support** - HTTPS with TLS 1.0-1.3 from PEM certificate and key files
- **Compression** - Automatic gzip/deflate based on Accept-Encoding
- **Keep-Alive** - Persistent connections with configurable limits
- **Form data** - URL-encoded and multipart form parsing with file uploads
- **Cookies** - Full request and response cookie support
- **Chunked encoding** - Streaming requests and responses with Transfer-Encoding: chunked

## Configuration

You can set various options on the server using the `with` methods:

```java
import java.time.Duration;

import org.lattejava.http.server.HTTPHandler;
import org.lattejava.http.server.HTTPListenerConfiguration;
import org.lattejava.http.server.HTTPServer;

public class Example {
  public static void main(String... args) {
    HTTPHandler handler = (req, res) -> {
      // Handler code goes here
    };

    HTTPServer server = new HTTPServer().withHandler(handler)
                                        .withShutdownDuration(Duration.ofSeconds(10L))
                                        .withListener(new HTTPListenerConfiguration(4242));
    server.start();
    // Use server
    server.close();
  }
}
```

## TLS

The HTTP server implements TLS 1.0-1.3 using the Java SSLEngine. To enable TLS, create an `HTTPListenerConfiguration` that includes a certificate and private key:

```java
import java.nio.file.Files;
import java.nio.file.Paths;

import org.lattejava.http.server.HTTPHandler;
import org.lattejava.http.server.HTTPListenerConfiguration;
import org.lattejava.http.server.HTTPServer;

public class Example {
  public static void main(String... args) throws Exception {
    String homeDir = System.getProperty("user.home");
    String certificate = Files.readString(Paths.get(homeDir + "/dev/certificates/example.org.pem"));
    String privateKey = Files.readString(Paths.get(homeDir + "/dev/certificates/example.org.key"));

    HTTPHandler handler = (req, res) -> {
      // Handler code goes here
    };

    HTTPServer server = new HTTPServer().withHandler(handler)
                                        .withListener(new HTTPListenerConfiguration(4242, certificate, privateKey));
    server.start();
    // Use server
    server.close();
  }
}
```

For development, you can use `mkcert` to create self-signed certificates:

```shell
brew install mkcert
mkcert -install
mkdir -p ~/dev/certificates
mkcert -cert-file ~/dev/certificates/example.org.pem -key-file ~/dev/certificates/example.org.key example.org
```

## Performance

A key goal of this project is screaming performance. Here are benchmark results comparing against other Java HTTP servers. All servers implement the same handler that reads the request body and returns a `200`.

### Baseline (100 concurrent connections)

| Server | Requests/sec | Avg latency (ms) | P99 latency (ms) | vs java-http |
|--------|-------------:|------------------:|------------------:|-------------:|
| java-http      |      114,483 |              0.86 |              1.68 |       100.0% |
| JDK HttpServer |       89,870 |              1.08 |              2.44 |        78.5% |
| Jetty          |      111,500 |              1.17 |             11.89 |        97.3% |
| Netty          |      117,119 |              0.85 |              1.75 |       102.3% |
| Apache Tomcat  |      102,030 |              0.94 |              2.41 |        89.1% |

### Under stress (1,000 concurrent connections)

| Server | Requests/sec | Failures/sec | Avg latency (ms) | P99 latency (ms) | vs java-http |
|--------|-------------:|-------------:|------------------:|------------------:|-------------:|
| java-http      |      114,120 |            0 |              8.68 |             11.88 |       100.0% |
| JDK HttpServer |       50,870 |      17655.7 |              6.19 |             22.61 |        44.5% |
| Jetty          |      108,434 |            0 |              9.20 |             14.83 |        95.0% |
| Netty          |      115,105 |            0 |              8.61 |             10.09 |       100.8% |
| Apache Tomcat  |       99,163 |            0 |              9.88 |             18.77 |        86.8% |

_Benchmark performed 2026-02-19 on Darwin, arm64, 10 cores, Apple M4, 24GB RAM (MacBook Air). Java: OpenJDK 21.0.10._
