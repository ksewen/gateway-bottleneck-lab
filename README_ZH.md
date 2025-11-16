# Gateway Bottleneck Lab

[English](./README.md) | [Deutsch](./README_DE.md)

> 基于一个经过脱敏与简化的真实生产问题。
> 本项目展示了为什么小粒度、聚焦的性能测试——特别是组件级和集成级的测试——能够帮助在早期发现隐藏的性能瓶颈，
> 从而避免在大型系统中产生复杂且高成本的问题。

## 🎯 项目目标

本仓库提供了一个**最小化**、**可复现**的测试环境，用于演示在典型的分布式架构
（Gateway → Authentication → Business Logic）中常见的性能瓶颈。

整个场景基于一个**真实**出现过的**生产问题**，但已经经过完全的**清理**、**脱敏**与**简化**，**不包含任何敏感信息**。

本项目的目的是帮助开发者理解：
- 为什么细粒度性能测试（从简单 **Benchmark** 到**组件级**、**集成级**性能测试）与系统级别的性能测试同样重要；
- 如何构建**可重复**、**可控**的性能测试环境，从而在**早期**发现潜在的性能问题。
- 
## 🧱 项目概览

本项目模拟了一条简化但接近真实的服务调用链，包括：

- **gateway-for-test** —— 网关
- **auth-service-for-test** —— 鉴权服务
- **service-for-test** —— 业务服务

所有组件均被完全**容器化**，**可独立运行**，也可以**通过 Docker 一键启动**整个环境。

在包含完整示例代码的分支中（例如 **0.0.1**、**0.0.2** 等），
借助**resources/** 目录下包含的 `docker-compose.yml`，可以一键启动完整的测试环境。

**main** 分支仅包含项目说明文档，不包含可运行代码。

架构示意图：

![Architektur](https://raw.githubusercontent.com/ksewen/Bilder/main/20251116160740231.png)

### 分支说明

各个分支代表系统的不同阶段与不同问题的复现状态：
- 分支 [**0.0.1**](https://github.com/ksewen/performance-test-example/tree/0.0.1)：包含一个刻意构造的性能瓶颈。
分析显示问题来源于：**在 WebFlux 中使用了基于 RestTemplate 的同步阻塞 HttpClient**，在高并发下会产生明显延迟。
- 分支 [**0.0.2**](https://github.com/ksewen/performance-test-example/tree/0.0.2)：修复了 [**0.0.1**](https://github.com/ksewen/performance-test-example/tree/0.0.1) 中的性能瓶颈，并对优化结果进行了评估。
在评估过程中进一步发现：**Logback 在输出日志时产生了可观察到的性能损耗**。
- 分支 [**0.0.3**](https://github.com/ksewen/performance-test-example/tree/0.0.3)：修复并分析了 [**0.0.2**](https://github.com/ksewen/performance-test-example/tree/0.0.2) 中发现的问题。
此外还包含了一个针对 **UUID 生成过程中随机数阻塞**问题的解决方案与详细说明。

每个分支都包含对应的测试方法、详细分析与测量结果。

## 🐳 使用 Docker 运行

### 构建 Docker 镜像

在项目根目录下，使用提供的脚本构建各服务镜像：

```shell
gateway-for-test/resources/scripts/build-image.sh -d .
service-for-test/resources/scripts/build-image.sh -d .
auth-service-for-test/resources/scripts/build-image.sh -d .
```

### 启动完整环境

```shell
cd resources && \
docker-compose --compatibility -f docker-compose.yml up
```

## 🧪 功能验证

Gateway 提供了一个简单的测试路由，可用于基础功能验证：

```shell
curl http://127.0.0.1:38071/service/hello
```
如需进一步测试（例如并发访问、模拟高压场景），可以使用外部工具模拟请求。
我个人选择了轻量级工具 [**wrk**](https://github.com/wg/wrk)￼，因为它简单、易复现，适用于本项目。

更完整的示例、测试脚本以及性能对比结果，请参考对应的分支（如 **0.0.1**、**0.0.2**）。