# Third Test Round

[Deutsch](./README_DE.md) | [简体中文](./README_ZH.md)

## 🎯 Goal

After the optimizations in the **second test round**, the system was tested again through the gateway.
The goals of this round were:

- to evaluate the effect of the **AsyncAppender**
- to find **other possible** hidden bottlenecks
- to understand differences between this lab setup and real production systems
- to check whether the system gets closer to the expected performance

## Throughput Test After the Change — Unexpected Result

After enabling the **AsyncAppender**, a new five‑minute throughput test was executed.

**Result**:

> Requests/sec: 3919.34

Original log:

```shell
Running 5m test @ http://127.0.0.1:38071/service/hello
  5 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.56ms  545.34us  29.72ms   92.53%
    Req/Sec   787.67     33.81     0.89k    77.66%
  Latency Distribution
     50%    2.48ms
     75%    2.69ms
     90%    2.94ms
     99%    4.09ms
  1176033 requests in 5.00m, 143.56MB read
Requests/sec:   3919.34
Transfer/sec:    489.92KB
```

The throughput was about 4% **lower** than in the second test round.

### Interpretation

Normally the AsyncAppender should reduce work on the request thread. But in this test setup with only **two CPU cores**,
the extra background thread caused more switching and small performance loss.

In a production system with **more CPU cores**, the result would likely be different.

## Deeper Analysis

### Reducing Log Output

The log level was changed to **ERROR**, to reduce I/O operations during each request.

This is also a common practice in real microservice environments.

**Result**:

> Requests/sec: 4369.11

Original log:

```shell
Running 5m test @ http://127.0.0.1:38071/service/hello
  5 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.31ms  761.12us  69.92ms   96.95%
    Req/Sec     0.88k    59.38     0.96k    85.50%
  Latency Distribution
     50%    2.21ms
     75%    2.36ms
     90%    2.56ms
     99%    4.36ms
  1311067 requests in 5.00m, 160.04MB read
Requests/sec:   4369.11
Transfer/sec:    546.14KB
```

The throughput **increased** by **about 6%**, matching the CPU overhead seen earlier. This confirms:

Lower log output has a **direct positive effect** on throughput.

### Analysis After Log Optimization

CPU and thread profiles were checked again to confirm that the gateway had no new bottlenecks.

![cpu-views-call-tree](https://github.com/ksewen/Bilder/blob/main/202308190029673.png?raw=true "CPU Views - Call Tree")

The results showed:

- no high amount of **BLOCK** or **WAIT**
- no signs of **event‑loop blocking**

So the gateway had **no obvious bottlenecks** after the log reduction.

## Further Explanation

### Improving Profiling Strategy — New Bottleneck Found

In earlier rounds, the JProfiler package filter was limited to `com.github.ksewen` to save time and resources.

To get a full system view, the filter was removed.
All libraries, frameworks and JVM internals became visible.
**Monitors & Locks** were also enabled.

This revealed a hidden bottleneck:

👉 **UUID generation**.

### UUID Bottleneck

In the filter
[TracingGlobalFilter](https://github.com/ksewen/gateway-bottleneck-lab/blob/0.0.3/gateway-for-test/src/main/java/com/github/ksewen/performance/test/gateway/filter/tracing/TracingGlobalFilter.java),
a UUID is created for every request using: `UUID.randomUUID()`

On Linux, this uses `SecureRandom`.
If the system uses `/dev/random` and the entropy is low, the call can **block**.

Profiling confirmed this problem:

![Blockierung-1](https://raw.githubusercontent.com/ksewen/Bilder/main/202308201438817.png)

![Blockierung-2](https://raw.githubusercontent.com/ksewen/Bilder/main/202308201439720.png)

![Statistik der Blockierung](https://raw.githubusercontent.com/ksewen/Bilder/main/202308201439000.png)

### Fix in This Lab

To avoid the blocking, the JVM was started with:

```shell
-Djava.security.egd=file:/dev/urandom
```

This makes `SecureRandom` use a **non‑blocking** entropy source.

### Java Version Difference (*JEP 273*)

The behavior also depends on the *Java version*:

- production system before: **Java 8**
- this lab: **Java 17**

Starting from **Java 9**, `SecureRandom` was changed according to *JEP 273*:  

📌 https://openjdk.org/jeps/273

Because of this change, the old performance bottleneck during UUID generation – which often happened under high load in **Java 8** – does not appear the same way **Java 17**.

**Java 17** already fix the core issue, so the **Java-8** bottleneck cannot be reproduced exactly in this project.
To keep the lab up to date, I do not downgrade to **Java 8**. Instead, the documentation explains why the behavior is different.

> A detailed Java‑8 analysis and a solution can be found here:
> 👉 [**uuid-benchmark**](https://github.com/ksewen/uuid-benchmark)

## 🟩 Summary

- The AsyncAppender showed **no improvement** and resulted in a **4% performance drop**.
- Reducing log output **increased** throughput by **6%**.
- After the log change, the gateway had **no clear bottlenecks**.
- A new bottleneck was found: **UUID generation**.
- The fix using `egd=/dev/urandom` works for **Java 17**, but behaves differently in **Java 8** (*JEP 273*).
- This round shows the importance of profiling strategy, log management, JVM knowledge, and system context.
- Performance is always **context‑dependent**, **methodical**, and **rarely perfectly reproducible**.
