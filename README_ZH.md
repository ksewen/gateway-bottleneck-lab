# 第一轮测试

## Backend Service 基准测试

在进行任何性能分析之前，**第一步**是确定核心服务的**最大吞吐能力**。  
只有明确了这一**基准值**，我们才能正确分析和解释后续测试（例如通过 Gateway 路由）中的性能差异。

本次基准测试使用 [wrk](https://github.com/wg/wrk) 执行以下命令：

```shell
wrk -t5 -c10 -d300s --timeout=10s --latency http://127.0.0.1:38072/hello
```

测试过程中，通过 docker stats 监控**系统资源**：

```shell
docker stats ${CONTAINER}
# 或将结果写入指定文件
docker stats ${CONTAINER} | awk '{print strftime("%m-%d-%Y %H:%M:%S",systime()), $0}' >> ${FILE_PATH_AND_NAME}
```

**结果**：

> Requests/sec: 12994.82

原始输出记录：

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

Backend 服务 CPU 使用率超过 **100%**，说明实例被完全压满。  
因此，Backend 的**基准吞吐能力约为 13,000 RPS**。

## Gateway 集成测试

**第二步**，通过 Gateway 调用下列路由进行测试：

```shell
wrk -t5 -c10 -d300s --timeout=10s --latency http://127.0.0.1:38071/service/hello
```

**结果**:

> Requests/sec: 1811.16

原始输出记录：

```shell
Running 5m test @ http://127.0.1:38071/service/hello
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

测试期间 CPU 使用情况：

- Gateway：**≈ 65%**
- Business Service：**≈ 20%**
- Authentication Service：**≈ 30%**

### 结果分析

当请求经过 Gateway 时，吞吐量从 **~13,000 RPS**（基准）下降至仅 **~1,800 RPS**。

同时，所有服务的 **CPU 使用率都不高**，说明 Gateway 可能存在内部的阻塞点，导致：

- 请求等待时间**上升  **
- Event-Loop **被拖慢  **
- 整体吞吐量**下降**  

因此，需要进一步深入分析具体原因。

## 为什么选择 wrk？

选择 wrk 的原因：

- 非常**轻量**，适合可重复性的性能测试
- **CPU 占用低**，不会干扰测试环境
- 提供**精确的延迟统计**（含 percentile）
- 非常**适合**定位性能瓶颈的**微基准测试**
- 快速模拟 **pipeline behaviour**

特别是在关注**吞吐量**与**延迟**分布的场景下，wrk 是非常合适的测试工具。

## 使用 JProfiler 进行深入分析

由于 Gateway 的 CPU 使用率上不去，因此需要借助 Profiling 工具。  
本次调试使用了 [JProfiler](https://www.ej-technologies.com/products/jprofiler/overview.html)，它提供：

- CPU Profiling  
- 线程分析  
- 调用树（Call Tree）分析  
- 堆（Heap）分析  

> 注意：  
> JProfiler 占用非常高的系统资源，因此不适用于生产环境，但非常适合构建可重复的本地性能测试场景。

### 配置

Gateway 的 Docker 镜像中已经内置 JProfiler Agent。  
连接方式如下：`127.0.0.1:38849`

为了快速定位问题并且潜在的问题很可能在这个位置，在 JProfiler 中添加了 Call Tree 过滤器：

```
com.github.ksewen
```

再次执行 wrk 测试并启动 Profiling，收集有效数据。

### 分析结果

调用树显示，方法：

`org.springframework.web.client.RestTemplate.postForEntity`

占用了最高的 CPU 时间：

![cpu-views-call-tree](https://raw.githubusercontent.com/ksewen/Bilder/main/202308161502917.png)

这说明：

❗ **在 WebFlux 的非阻塞模型中，代码使用了同步阻塞的 RestTemplate（基于 HttpClient）。**

这种写法会导致：

- 线程阻塞  
- Reactor 线程上下文切换  
- Worker 池被拖慢  
- 延迟急剧上升  
- 吞吐量严重下降

## 问题修复

正确的解决方式是将阻塞式 RestTemplate 全量替换为 **WebClient**，因为 WebClient：

- 使用**非阻塞 I/O（Reactor Netty）**
- 原生支持 WebFlux
- 不会阻塞 Event-Loop

完成替换后，系统**有可能**达到：

- **更低**的延迟  
- **更高**的吞吐量

## 🟩 总结

本轮测试的主要发现如下：

- **基准性能约 13k RPS**，Backend 表现正常。  
- **集成 Gateway 后吞吐仅约 1.8k RPS**，说明存在严重内部瓶颈。  
- JProfiler 确认问题来自于 **RestTemplate 的阻塞调用**。  
- 初步方案是：切换到 **WebClient**（非阻塞），其效果会在后续分支中验证。
