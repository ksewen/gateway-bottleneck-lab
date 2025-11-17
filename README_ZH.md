# 第三轮测试

[English](./README.md) | [Deutsch](./README_DE.md)

## 🎯 目标

在完成**第二轮**测试的优化之后，需要再次验证系统在通过 Gateway 时的整体表现。

本轮的目标包括：

- 真实评估 AsyncAppender 的**实际效果**
- 继续发现**潜在的隐藏性能瓶颈**
- 深入理解实验环境与真实生产环境的差异
- 检查系统在前两轮优化后是否更接近预期性能表现

## 启用 AsyncAppender 后的吞吐测试（以及意外结果）

启用 **AsyncAppender** 后，再次执行 5 分钟的吞吐测试。

**测试结果**：

> Requests/sec: 3919.34

原始输出：

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

与第二轮相比，吞吐量**下降**了约 4 %。

### 解释

理论上 AsyncAppender 应该减少请求线程的压力，但在当前环境下性能却 **略微降低**。

原因很可能是实验环境只有 **两个** CPU 核心。日志异步线程会引发额外的上下文切换，导致收益不足以覆盖开销。

在拥有**更多 CPU 核心的生产环境**中，结果可能会完全不同。

## 深度分析

### 降级日志输出

将日志级别调整为 **ERROR**，尽量减少请求路径上的 I/O 开销。

这种方式在真实微服务集群中非常常见，因为现代平台都支持运行时**动态调整日志级别**。

降级日志输出后再次进行测试。

**测试结果**：

> Requests/sec: 4369.11

原始输出：


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

吞吐量**提升**了约 **6 %**。

这个提升与上一轮中观测到的日志 CPU 开销相符，说明：

降级日志输出会直接**提升**吞吐量。

### 优化日志输出后的分析

再次检查 CPU 和线程状态：

![cpu-views-call-tree](https://github.com/ksewen/Bilder/blob/main/202308190029673.png?raw=true "CPU Views - Call Tree")

结果显示：
- 没有明显的 **BLOCK** 或 **WAIT**
- 没有出现 event‑loop 被**阻塞**的迹象

因此，日志优化后 Gateway 本身**不再存在明显瓶颈**。

## 更多说明

### 调整分析策略——发现新的瓶颈

之前的几轮测试中，JProfiler 的包过滤器只关注
`com.github.ksewen`
以节省时间和资源。

为了进行更完整的分析，本轮将过滤器全部移除，并启用 Monitors & Locks 监控。

这样才能看到完整的请求路径，包括所有：

- 框架代码
- 库代码
- JVM 内部操作

在这种更完整的视角下，一个新的问题被发现了：

👉 **UUID 生成过程产生了阻塞**

### UUID 生成过程的阻塞

在
[TracingGlobalFilter](https://github.com/ksewen/gateway-bottleneck-lab/blob/0.0.3/gateway-for-test/src/main/java/com/github/ksewen/performance/test/gateway/filter/tracing/TracingGlobalFilter.java)
中，每次请求都会调用：`UUID.randomUUID()`

在 Linux 中，这依赖 `SecureRandom`。
如果操作系统使用 `/dev/random` 且熵不足，会出现**阻塞**。

Profiling 清晰显示了这一点：

![Blockierung-1](https://raw.githubusercontent.com/ksewen/Bilder/main/202308201438817.png)

![Blockierung-2](https://raw.githubusercontent.com/ksewen/Bilder/main/202308201439720.png)

![Statistik der Blockierung](https://raw.githubusercontent.com/ksewen/Bilder/main/202308201439000.png)

### 本次实验的解决方式

添加 JVM 启动参数：

```shell
-Djava.security.egd=file:/dev/urandom
```

让 `SecureRandom` 使用**非阻塞熵源**，从而避免 UUID 生成卡顿。

### 与 Java 版本相关（*JEP 273*）

UUID 阻塞与 **Java 版本**高度相关：

- 原来的生产环境：**Java 8**
- 本实验使用：**Java 17**

从 **Java 9** 开始，`SecureRandom` 根据 *JEP 273* 进行了重新设计：

📌 https://openjdk.org/jeps/273

因此 **Java 8** 与 **Java 17** 在 UUID 生成行为上存在显著差异。

> 关于 **Java 8 的详细分析与解决方案**，请参考我的另一个项目：  
> 👉 [**uuid-benchmark**](https://github.com/ksewen/uuid-benchmark)

## 🟩 总结

- AsyncAppender 在本轮测试中**没有提升性能**，反而造成 **4 % 的下降**。
- 将日志级别改为 ERROR 后，吞吐量**提高了 6 %**。
- Gateway 在优化后**未再发现明显瓶颈**。
- 取消包过滤后发现新的瓶颈：**UUID 生成阻塞**。
- `egd=/dev/urandom` 的方式在 **Java 17** 环境有效，但行为与 **Java 8** 不同（见 *JEP 273*）。
- 这一轮清晰展示了：Profiling 策略、日志管理、JVM 知识与环境差异，都会直接影响性能分析结果。
- 性能分析永远是 **依赖上下文**、**依赖方法论**，**并且难以完全复现的**。
