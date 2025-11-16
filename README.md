# Gateway Bottleneck Lab

[Deutsch](./README_DE.md) | [简体中文](./README_ZH.md)

> Based on a real production issue (simplified). It shows why doing small performance tests — especially at the
> component level (like integration tests) — helps find hidden bottlenecks early and avoids costly problems in large
> systems.

## 🎯 Purpose of the Project

This repository provides a **minimal reproducible environment** that shows typical performance problems in a distributed
architecture (Gateway → Authentication → Business Logic).

The scenario is based on a **real production issue**, but it was fully **cleaned**, **simplified**, and contains **no
confidential data**.

This project helps developers better understand:

- why fine-grained performance tests – from simple **benchmarking** to **component** or **integration** tests– are as
  important as
  **system-wide performance** tests.
- how to build **reproducible test environments** to find performance issues **early**.

## 🧱 Project Overview

The project simulates a **simple** but **realistic** service chain:

- **gateway-for-test** – API Gateway
- **auth-service-for-test** – Authentication Service
- **service-for-test** – Business Service

All services are fully **containerized** and can **run alone** or **together with Docker**.

In the branches that contain the full example code (for example **0.0.1**, **0.0.2**, etc.), there is a
`docker-compose.yml` file inside the **resources/** folder. This file allows you to start the whole test environment
quickly.

The **main** branch contains only the project description and no runnable code.

A simple architecture diagram:

![Architektur](https://raw.githubusercontent.com/ksewen/Bilder/main/20251116160740231.png)

### Branches

Different branches show different versions of the system:

- Branch [**0.0.1**](https://github.com/ksewen/performance-test-example/tree/0.0.1) contains a planned performance
  bottleneck. The problem comes from using a **RestTemplate with a synchronous HttpClient inside WebFlux**, which
  creates high latency under load.
- Branch [**0.0.2**](https://github.com/ksewen/performance-test-example/tree/0.0.2) fixes the bottleneck from
  [**0.0.1**](https://github.com/ksewen/performance-test-example/tree/0.0.1) and
  includes an evaluation of the improvements. During testing, it also shows that **Logback creates noticeable
  performance cost while writing logs**.
- Branch [**0.0.3**](https://github.com/ksewen/performance-test-example/tree/0.0.3) fixes and analyses the issues found
  in [**0.0.2**](https://github.com/ksewen/performance-test-example/tree/0.0.2).
  It also includes a solution and details for the **blocking issue during UUID generation**.

Each branch contains more detailed explanations, test methods, and results.

## 🐳 Running with Docker

### Build Docker Images

In the project root directory, you can build all service images using the provided scripts:

```shell
gateway-for-test/resources/scripts/build-image.sh -d .
service-for-test/resources/scripts/build-image.sh -d .
auth-service-for-test/resources/scripts/build-image.sh -d .
```

### Start the Environment

```shell
cd resources && \
docker-compose --compatibility -f docker-compose.yml up
```

## 🧪 Functional Test

The Gateway provides a simple test route. You can call it like this:

```shell
curl http://127.0.0.1:38071/service/hello
```

For more tests, you can also use external tools to create parallel or high-load requests.
I use a lightweight tool like [**wrk**](https://github.com/wg/wrk), because it is good for simple and reproducible load
tests.

More detailed examples and results can be found in the related branches (for example **0.0.1**, **0.0.2**).
