# 第二轮测试

[English](./README.md) | [Deutsch](./README_DE.md)

## 🎯 目标

在第一轮测试中（Gateway 内部使用 RestTemplate 造成阻塞）的问题得到修复之后，需要再次确认系统在与 Gateway 集成后整体吞吐能力。

本轮测试的目标是：

- **测量**切换到 WebClient 之后的实际性能变化
- 找出可能存在的**新的性能瓶颈**
- 为后续优化建立新的性能基线

## 修复后的集成测试

在将 Gateway 内部调用替换为 WebClient 之后，再次进行了 5 分钟的吞吐测试：

```shell
wrk -t5 -c10 -d300s --timeout=10s --latency http://127.0.0.1:38071/service/hello
```

**测试结果**：

> Requests/sec: 4091.91

原始输出：

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

### 结果解读

- 吞吐量从 **约 1,800 RPS** 提升到 **约 4,091 RPS**
- 提升幅度超过 **125%**
- 延迟表现也有明显改善

尽管性能有了大幅提升，但仍未接近基线值（约 13,000 RPS）。 因此需要继续分析剩余的潜在瓶颈。

## 深入分析

和第一轮一样，本轮依然使用
[JProfiler](https://www.ej-technologies.com/jprofiler)
对 Gateway 进行 CPU 调用分析。

一次 CPU Snapshot 显示，
**大量 CPU 时间消耗在日志输出（Logback）上**：

![cpu-views-call-tree](https://raw.githubusercontent.com/ksewen/Bilder/main/202308161502704.png)

### 为什么日志输出会成为新的瓶颈？

Logback 默认采用 **同步日志输出**，意味着业务线程必须等待日志写入完成。

在高并发场景下，这会导致：

- 在关键执行路径上的阻塞
- 不必要的 I/O 等待
- Event Loop 被拖慢
- 系统整体吞吐下降

结论很明显：  
**在修复了第一个瓶颈之后，同步日志输出成为新的主要性能限制因素。**

## 问题修复方向

Logback 提供了 [AsyncAppender](https://logback.qos.ch/manual/appenders.html#AsyncAppender)，可以将日志输出放到后台线程异步处理。

AsyncAppender 的优势：

- **减轻**业务线程的负担
- 日志**异步**写入，不阻塞主流程
- 大幅**减少**关键路径上的**阻塞**
- 在高并发场景下扩展性更好

典型配置示例：

```xml

<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="FILE"/>
</appender>
```

这样日志写入就从主逻辑中脱离出来，在合适的条件下能显著改善请求处理速度。

## 预期结果

启用 AsyncAppender 后，预期将会：

- 延迟进一步**降低**
- 吞吐量进一步**提升**
- Gateway 中的阻塞时间进一步**减少**

最终的实际效果将在 
[**下一轮测试**](https://github.com/ksewen/gateway-bottleneck-lab/tree/0.0.3?tab=readme-ov-file) 
进行验证。

## 🟩 总结

- 第一次修复（WebClient 替换 RestTemplate）非常成功：吞吐量提升 **125%**
- 使用 JProfiler 发现了新的瓶颈：**同步日志写入**
- Logback 的 **AsyncAppender** 是一个可能的优化方向，实际效果会在下一轮测试中验证
