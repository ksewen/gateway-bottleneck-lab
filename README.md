# The First Test Round

[Deutsch](./README_DE.md) | [简体中文](./README_ZH.md)

## Baseline Benchmark of the Backend Service

**The first step** of any performance analysis is to check the **maximum throughput** of one single core service.  
Only with this baseline value we can understand later differences, for example when the request goes through the
Gateway.

To measure the baseline performance, the following command was used with [wrk](https://github.com/wg/wrk):

```shell
wrk -t5 -c10 -d300s --timeout=10s --latency http://127.0.0.1:38072/hello
```

During the test, the **system resources** were monitored via:

```shell
docker stats ${CONTAINER}
# or write the values to a file:
docker stats ${CONTAINER} | awk '{print strftime("%m-%d-%Y %H:%M:%S",systime()), $0}' >> ${FILE_PATH_AND_NAME}
```

**Result:**

> Requests/sec: 12994.82

Original protocol:

```shell
Running 5m test @ http://127.0.0.1:38072/hello
  5 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     3.08ms    6.10ms  52.91ms   87.33%
    Req/Sec     2.61k   219.70     3.65k    79.65%
  Latency Distribution
     50%  583.00us
     75%  779.00us
     90%   12.39ms
     99%   26.01ms
  3899171 requests in 5.00m, 469.24MB read
Requests/sec:  12994.82
Transfer/sec:      1.56MB
```

The backend CPU usage was **over 100%**, which means the instance was fully used.  
So the baseline performance is **~13,000 RPS**.


## Throughput Test Through the Gateway

**In the second step**, this route was tested through the Gateway:

```shell
wrk -t5 -c10 -d300s --timeout=10s --latency http://127.0.0.1:38071/service/hello
```

**Result:**

> Requests/sec: 1811.16

Original protocol:

```shell
Running 5m test @ http://127.0.0.1:38071/service/hello
  5 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     6.29ms    6.75ms 206.86ms   97.07%
    Req/Sec   363.95     67.84   545.00     85.73%
  Latency Distribution
     50%    5.17ms
     75%    6.52ms
     90%    8.22ms
     99%   43.07ms
  543438 requests in 5.00m, 66.34MB read
Requests/sec:   1811.16
Transfer/sec:    226.39KB
```

CPU usage during the test:

- Gateway: **~65%**
- Business Service: **~20%**
- Authentication Service: **~30%**

### Interpretation

The throughput drops from **~13,000 RPS** (baseline) to only **~1,800 RPS** when the request goes through the Gateway.

At the same time, all services have **low CPU usage**, which means the Gateway may have an internal blocking problem:

- **long** waiting times
- **slow** event-loop
- **lower** overall throughput

To find the real cause, a deeper analysis is needed.

## Why wrk

wrk was used because it is:

- **very lightweight** (good for repeatable tests)
- has **very low CPU overhead**
- gives **accurate latency statistics**
- **good for micro-benchmarks** that focus on bottlenecks
- useful when testing **pipeline behaviour**

Because this project focuses on **throughput** and **latency** distribution, wrk is a perfect fit without adding too
much overhead.

## Detailed Analysis with JProfiler

Because the CPU usage of the Gateway did not rise much, profiling was needed.  
For this part, [JProfiler](https://www.ej-technologies.com/products/jprofiler/overview.html) was used.  
JProfiler supports:

- CPU profiling
- thread profiling
- call tree analysis
- heap analysis

> Note:  
> JProfiler has too much overhead for production,  
> but is ideal for a reproducible test setup like this project.

### Configuration

The JProfiler agent was already added in the Gateway Docker image.  
The connection was made through: `127.0.0.1:38849`

A call tree filter was added for the package `com.github.ksewen`, because the possible issue was expected there.

During profiling, another wrk test was executed to collect useful data.

### Analysis

The call tree clearly shows that the method:

`org.springframework.web.client.RestTemplate.postForEntity`

uses most of the CPU time.

![cpu-views-call-tree](https://raw.githubusercontent.com/ksewen/Bilder/main/202308161502917.png)

This confirms the problem:

❗ **In a reactive WebFlux environment, a synchronous and blocking RestTemplate with HttpClient was used.**

This leads to:

- blocked threads
- context switching
- slow worker pools
- high latency
- very low throughput

## Fix

The correct solution is to replace the blocking RestTemplate with **WebClient**, because it:

- uses **non-blocking I/O (Reactor Netty)**
- is designed for WebFlux
- does not block the event-loop

With this change, the system could reach:

- much **lower latency**
- much **higher throughput**

## 🟩 Summary

The first test round shows:

- The **baseline benchmark (~13k RPS)** proves that the backend works very well.
- The **gateway throughput test (~1.8k RPS)** shows a strong internal bottleneck.
- JProfiler confirms the **blocking RestTemplate** call as the cause.
- A possible next step is to **switch to WebClient** (reactive / non-blocking).  
  The effect will be checked in the next branches.
