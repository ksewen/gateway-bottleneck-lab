# Second Test Round

[Deutsch](./README_DE.md) | [简体中文](./README_ZH.md)

## 🎯 Goal

After the bottleneck from the first test round (a blocking RestTemplate call in the Gateway) was fixed, the system
throughput through the Gateway needed to be tested again.

The goals of this round were:

- to **measure the effect** of switching to WebClient
- to find any new **possible bottlenecks**
- to create a new baseline for later optimizations

## Throughput Test After the Fix

After switching to WebClient, a new five‑minute throughput test was run through the Gateway:

```shell
wrk -t5 -c10 -d300s --timeout=10s --latency http://127.0.0.1:38071/service/hello
```

**Result**:

> Requests/sec: 4091.91

Original log:

```shell
Running 5m test @ http://127.0.0.1:38071/service/hello
  5 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.47ms  772.41us  43.05ms   96.34%
    Req/Sec   822.41     55.76     0.92k    86.56%
  Latency Distribution
     50%    2.36ms
     75%    2.54ms
     90%    2.80ms
     99%    4.65ms
  1227963 requests in 5.00m, 149.90MB read
Requests/sec:   4091.91
Transfer/sec:    511.49KB
```

### Interpretation

- Throughput increased from **~1,800 RPS** (before the fix) to **~4,091 RPS**.
- This is an improvement of **over 125%**.
- Latency values also **improved clearly**.

Even with this improvement, the expected value (closer to the baseline of ~13,000 RPS) was not reached.  
So a deeper analysis was needed to find the next bottleneck.

## Deeper Analysis

As in the first test round, profiling was done again using  
[JProfiler](https://www.ej-technologies.com/jprofiler).

A CPU snapshot showed that **a large part of the execution time was used for log output**:

![cpu-views-call-tree](https://raw.githubusercontent.com/ksewen/Bilder/main/202308161502704.png)

### Why does this bottleneck happen?

Logback writes logs **synchronously by default**. This means the working thread must wait until the log entry is
written.

Under high load, this can cause:

- blocking in the critical request path
- unnecessary I/O waiting
- slowdown of the event loop
- lower total throughput

So it became clear:  
**After the first fix, synchronous log output was the next important performance factor.**

## Fix

Logback provides an [AsyncAppender](https://logback.qos.ch/manual/appenders.html#AsyncAppender), which moves log writing
into background threads.

Advantages of the AsyncAppender:

- **reduces load** on the working thread
- logs are written **asynchronously**
- **fewer blockings** in the request flow
- better scaling under high load

Example:

```xml

<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="FILE"/>
</appender>
```

With this change, log writing moves out of the main logic, and request handling should become faster.

## Expected Result

After enabling the AsyncAppender, it is expected that:

- latency becomes **lower**
- throughput becomes **higher**
- blocking time in the Gateway becomes **smaller**

The real effect will be checked in the 
[**next test round**](https://github.com/ksewen/gateway-bottleneck-lab/tree/0.0.3?tab=readme-ov-file).

## 🟩 Final Summary

- The first fix (WebClient) was successful: **+125% throughput**.
- JProfiler showed that **synchronous log output** became the next bottleneck.
- Logback’s **AsyncAppender** offers a possible solution. Its real effect will be tested in the next round.
