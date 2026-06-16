---
layout: docs
title: Performance
description: JMH benchmarks comparing latte json against Jackson and Gson.
weight: 90
---

Because the serializer and deserializer are generated at compile time, there's no reflection, no per-call type introspection, and no schema cache to warm up — the companion is straight-line Java that reads and writes fields directly. The result is throughput at or above Jackson's, with markedly lower allocation, and a comfortable lead over Gson.

## Benchmarks

Measured with JMH against Jackson databind and Gson across six workloads, in both directions. Higher ops/sec is better; lower allocation (bytes per operation, in parentheses) is better.

- MacBook Air (Apple M4), OpenJDK 25.0.2
- Versions: latte 0.4.0, jackson 2.19.0, gson 2.14.0, JMH 1.37

### Serialize

| Scenario | latte                | jackson               | gson                  |
|----------|----------------------|-----------------------|-----------------------|
| api      | 702,250 (1640 B/op)  | 602,511 (3048 B/op)   | 345,743 (10456 B/op)  |
| deep     | 1,221,873 (848 B/op) | 1,140,479 (1904 B/op) | 634,186 (4976 B/op)   |
| jwt      | 4,568,832 (232 B/op) | 4,069,549 (760 B/op)  | 2,387,330 (1560 B/op) |
| large    | 8,781 (110120 B/op)  | 8,990 (225407 B/op)   | 4,421 (587745 B/op)   |
| numbers  | 65,491 (13176 B/op)  | 63,743 (50949 B/op)   | 36,744 (98372 B/op)   |
| strings  | 140,574 (9992 B/op)  | 137,004 (19197 B/op)  | 50,039 (131952 B/op)  |

### Deserialize

| Scenario | latte                 | jackson               | gson                  |
|----------|-----------------------|-----------------------|-----------------------|
| api      | 311,358 (11984 B/op)  | 261,740 (12736 B/op)  | 221,359 (19072 B/op)  |
| deep     | 1,502,094 (1728 B/op) | 753,220 (4016 B/op)   | 539,450 (7296 B/op)   |
| jwt      | 5,005,977 (672 B/op)  | 3,217,211 (1472 B/op) | 1,646,201 (3904 B/op) |
| large    | 7,370 (343848 B/op)   | 5,764 (344745 B/op)   | 3,105 (772466 B/op)   |
| numbers  | 29,713 (85872 B/op)   | 23,186 (97552 B/op)   | 14,811 (147728 B/op)  |
| strings  | 161,630 (21360 B/op)  | 186,747 (22112 B/op)  | 69,609 (82856 B/op)   |

The workloads span a flat JWT-style claims object (`jwt`), a typical mixed-type API resource (`api`), a 1,000-record list (`large`), escape- and unicode-heavy text (`strings`), `long`/`BigDecimal`-heavy data (`numbers`), and a graph nested to depth 14 (`deep`).

## Methodology

All three libraries bind the same record classes and produce/consume `byte[]` — the real boundary for network, disk, and JWT payloads. Jackson runs with a shared `ObjectMapper` (JSR-310 module, `Instant` as ISO-8601, non-null inclusion); Gson runs with a shared instance and an `Instant` adapter, and its String API includes the UTF-8 conversion that boundary genuinely costs. Each benchmark runs in its own forked JVM so the libraries never share JIT profiles, and every payload is round-tripped through every library and compared for semantic equality before any timing counts.

JMH numbers move a few percent run-to-run with machine load, so treat them as ratios on an idle machine rather than absolutes. The full harness — benchmark sources, payloads, and the orchestration scripts — lives in [`benchmarks/`](https://github.com/latte-java/json/tree/main/benchmarks) in the repository.
