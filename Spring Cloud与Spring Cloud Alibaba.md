# Spring Cloud 与 Spring Cloud Alibaba — 面试备战手册

> 内容来源：Spring 官方文档、Spring Cloud Alibaba 官方文档、Spring 官方 GitHub
> Spring Cloud = 微服务的一站式解决方案，是微服务架构的标准规范
> Spring Cloud Alibaba = 这个规范的一套实现（用阿里组件替换 Netflix 停更组件）

---

## 📌 一、从 Spring Boot 到 Spring Cloud：增加了什么？

### Spring Boot 解决了什么？

```
Spring Boot = Spring + 自动配置 + 嵌入式服务器 + 生产级监控
```

它让单个微服务应用的搭建变得极其简单——一个 `@SpringBootApplication` + 一个 `application.yml` 就能跑起来。

### Spring Cloud 解决了什么？

单个服务跑起来后，多个服务之间产生了**分布式系统的问题**：

```
服务A（端口8081）→ 怎么找到服务B（端口8082）？
                  → 调用失败了怎么办？
                  → 配置改了怎么同时通知所有服务？
                  → 调用链跨越5个服务，怎么排查慢在哪？
```

**这些问题是 Spring Boot 解决不了的**，Spring Cloud 就是为解决这些问题而生：

| 微服务面临的问题 | Spring Cloud 的解法 |
|-----------------|-------------------|
| 服务A怎么找到服务B？ | **服务注册与发现**（Eureka/Nacos） |
| 调用多个实例时怎么分配？ | **客户端负载均衡**（Ribbon/LoadBalancer） |
| 服务B挂了怎么办？ | **熔断降级**（Hystrix/Sentinel） |
| 配置分散在各服务怎么管理？ | **配置中心**（Config/Nacos） |
| 外部请求怎么统一入口？ | **API网关**（Zuul/Gateway） |
| 跨服务调用怎么简化？ | **声明式调用**（Feign/OpenFeign） |
| 调用链太长怎么排查？ | **分布式追踪**（Sleuth/Zipkin/Micrometer） |
| 配置变更怎么通知所有服务？ | **消息总线**（Bus） |

### 一句话总结

```
Spring Boot       = 让单个微服务跑起来
Spring Cloud      = 让一群微服务协同工作
Spring Cloud Alibaba = 用阿里系组件替换已停更的 Netflix 组件
```

---

## 📌 二、三大体系对照

### 组件全览

| 功能模块 | Spring Cloud Netflix（传统/已停更） | Spring Cloud Alibaba（阿里系） | Spring Cloud 官方（新体系） |
|---------|-----------------------------------|-------------------------------|--------------------------|
| **服务注册发现** | ✅ Eureka（AP架构，已停止更新） | ✅ **Nacos**（AP/CP可切换，**推荐**） | ✅ Consul / K8s |
| **配置中心** | ✅ Spring Cloud Config + Bus（需Git） | ✅ **Nacos Config**（**自带控制台**） | ✅ Consul |
| **负载均衡** | ✅ Ribbon（已停止更新） | ✅ **Spring Cloud LoadBalancer**（官方推荐） | ✅ LoadBalancer |
| **熔断降级** | ✅ Hystrix（已停止更新） | ✅ **Sentinel**（**有控制台、实时规则**） | ✅ Circuit Breaker + Resilience4j |
| **API网关** | ✅ Zuul 1.x（阻塞IO，性能差） | ✅ **Spring Cloud Gateway**（**非阻塞**） | ✅ Gateway |
| **声明式调用** | ✅ Feign | ✅ **OpenFeign**（开源社区维护） | ✅ OpenFeign |
| **分布式追踪** | ✅ Sleuth + Zipkin | ✅ Sleuth + Zipkin / **Micrometer** | ✅ Micrometer Tracing |
| **消息驱动** | ✅ Stream | ✅ **Stream + RocketMQ** | ✅ Stream |

### 核心差异速览

```
Spring Cloud Netflix（传统）     Spring Cloud Alibaba（阿里系）
┌────────────────────┐        ┌────────────────────┐
│    Eureka          │   →    │    Nacos           │ ← 注册中心+配置中心二合一
│    Ribbon          │   →    │    LoadBalancer    │ ← 官方新组件
│    Hystrix         │   →    │    Sentinel        │ ← 有控制台，实时规则配置
│    Zuul            │   →    │    Gateway         │ ← 异步非阻塞，性能好5倍
│    Config + Bus    │   →    │    Nacos Config    │ ← 直接支持配置管理
│    Stream + Kafka  │   →    │    Stream + RMQ    │ ← 支持RocketMQ
└────────────────────┘        └────────────────────┘
```

---

## 📌 三、各组件详解

### 1. 服务注册与发现：Eureka vs Nacos

#### Eureka（Netflix 体系）

```
服务启动 → 注册到 Eureka Server → 每30秒发心跳
服务调用 → 从 Eureka 拉取服务列表 → Ribbon 负载均衡 → 调用目标
```

**核心特点**：
- **AP 架构**（可用性优先、最终一致性）
- 自我保护模式：网络抖动时不轻易剔除实例，宁可保留过期数据也不能让服务不可用
- **已停止更新**（Netflix 2018 年宣布维护模式）

#### Nacos（Alibaba 体系）

```
Nacos = Naming + Configuration = 注册中心 + 配置中心（二合一）

相比于 Eureka 的优势：
① 支持 AP/CP 模式切换（默认 AP，需要时可切 CP）
② 内置配置中心，不用额外部署 Config Server
③ 有 Web 控制台，可视化管理服务和配置
④ 健康检查更可靠（主动探测 + 客户端心跳）
⑤ 支持环境隔离、分组、权重路由
```

**面试回答**：

> **Q：Nacos 相比 Eureka 强在哪？**
>
> Nacos 和服务发现和配置中心二合一，一台 Nacos 就能替代 Eureka + Config Server + Bus 三套组件。其次 Nacos 支持 AP/CP 模式切换，关键时刻可以改为 CP 模式保证数据一致性。还有 Nacos 有自带的可视化控制台，可以在网页上直接管理服务和配置，而 Eureka 需要另外搭配套。

---

### 2. 配置中心：Spring Cloud Config vs Nacos Config

#### Spring Cloud Config

```
配置文件存 Git → Config Server 读取 → Config Client 启动时拉取
配置变更：改 Git → 手动调用 /actuator/refresh → 只刷新当前服务
全量刷新：需要配合 Spring Cloud Bus + 消息队列 广播刷新事件
```

**缺点**：
- 需要额外部署 Config Server、Git 仓库、Bus、消息队列
- 配置刷新需要主动调用 actuator 端点，不是实时推送
- 没有可视化界面

#### Nacos Config

```
配置直接写在 Nacos 控制台 → 客户端长轮询监听 → 配置变更自动推送
```

**优势**：
- **实时推送**：基于长轮询机制，配置改了服务马上感知（不需要手动 refresh）
- **内置控制台**：Web 界面管理配置，支持版本回滚、灰度发布、权限控制
- **轻量**：不需要 Git、Bus、消息队列，一台 Nacos 搞定

---

### 3. 熔断降级：Hystrix vs Sentinel

#### Hystrix

**工作模式**：线程池隔离（每个依赖一个线程池）或信号量隔离

**三个状态**：CLOSED（正常） → OPEN（熔断） → HALF_OPEN（试探恢复）

**缺点**：
- 线程池隔离有额外线程切换开销
- 控制台需要 Hystrix Dashboard + Turbine 聚合，配置修改需重启
- **已停止更新**

#### Sentinel

| 对比项 | Hystrix | Sentinel |
|--------|---------|----------|
| **隔离方式** | 线程池/信号量 | 信号量（轻量） |
| **限流维度** | 简单 | QPS/线程数/关联/链路/预热排队 |
| **熔断策略** | 异常比例 | 慢调用比例/异常比例/异常数 |
| **规则配置** | 改代码重启 | 控制台**实时推**，无需重启 |
| **控制台** | 需额外搭建 | **开箱即用** |

**面试回答**：

> **Q：Sentinel 比 Hystrix 强在哪？**
>
> Sentinel 是阿里双11 高并发场景下打磨出来的。比 Hystrix 多了限流能力（Hystrix 主要做熔断，限流比较弱）——支持 QPS 限流、线程数限流、关联限流、预热排队等多种方式。其次 Sentinel 有独立的可视化控制台，限流熔断规则改了能实时推送上去，不需要重启服务。Hystrix 改配置要改代码重新部署。

---

### 4. API 网关：Zuul vs Gateway

#### Zuul 1.x（Netflix）

```
每个请求 → 分配一个线程处理（Servlet 阻塞模型）
连接数 = 线程数 → 连接多了线程数暴涨 → 上下文切换开销大
```

**瓶颈**：阻塞 IO，IO 操作（读请求体、写响应）时线程被挂起，大量连接时性能急剧下降。

#### Spring Cloud Gateway

```
事件驱动（WebFlux 非阻塞模型）
少量线程即可处理大量连接
性能比 Zuul 高 5 倍以上
```

**核心特性**：
- 路由规则用 `RouteLocator` 配置：根据 Path、Header、Query 等条件匹配
- 过滤器链：`GlobalFilter` + `GatewayFilter`，支持前置过滤和后置过滤
- 内置限流（RequestRateLimiter）、熔断（CircuitBreaker）、重试

---

### 5. 声明式调用：Feign / OpenFeign

```java
// 只需一个接口 + 注解，无需写 HTTP 调用代码
@FeignClient(name = "order-service")
public interface OrderClient {
    @GetMapping("/order/{id}")
    Order getOrder(@PathVariable Long id);
}
```

**Feign**（已并入 OpenFeign）：整合了 Ribbon 负载均衡 + Hystrix 熔断。
**OpenFeign**（社区维护）：底层支持 OkHttp、HttpClient，性能更好。

---

## 📌 四、架构全图对比

### Spring Cloud Netflix 传统架构

```
                          ┌─────────────┐
                          │  Zuul 网关   │ ← 入口：路由/鉴权/限流
                          └──────┬──────┘
                                 │
           ┌─────────────────────┼─────────────────────┐
           ▼                     ▼                     ▼
     ┌──────────┐          ┌──────────┐          ┌──────────┐
     │ 服务A    │          │ 服务B    │          │ 服务C    │
     │(Feign+Rib│          │(Feign+Rib│          │(Feign+Rib│
     │bon+Hystr│          │bon+Hystr│          │bon+Hystr│
     │ix)      │          │ix)      │          │ix)      │
     └────┬─────┘          └────┬─────┘          └────┬─────┘
          │                     │                     │
          └──────────┬──────────┴──────────┬──────────┘
                     │                     │
               ┌─────▼─────┐         ┌─────▼─────┐
               │  Eureka   │         │  Config   │
               │ (注册中心) │         │ (配置中心) │
               └───────────┘         └───────────┘
                             还需要 Bus + MQ 做配置刷新
```

### Spring Cloud Alibaba 架构

```
                          ┌────────────────┐
                          │  Gateway 网关   │ ← 非阻塞，性能高
                          └───────┬────────┘
                                  │
           ┌──────────────────────┼──────────────────────┐
           ▼                      ▼                      ▼
     ┌──────────┐           ┌──────────┐           ┌──────────┐
     │ 服务A    │           │ 服务B    │           │ 服务C    │
     │(OpenFeign│           │(OpenFeign│           │(OpenFeign│
     │+LB+Sent│           │+LB+Sent│           │+LB+Sent│
     │inel)    │           │inel)    │           │inel)    │
     └────┬─────┘           └────┬─────┘           └────┬─────┘
          │                      │                      │
          └──────────┬───────────┴───────────┬──────────┘
                     │                      │
               ┌─────▼──────┐          ┌─────▼──────┐
               │   Nacos    │          │   Nacos    │
               │ (注册中心)   │ ← 同一个 Nacos ──┤ (配置中心)  │
               └────────────┘          └────────────┘
```

**关键差异**：Netflix 体系需要 5-6 个组件（Eureka + Config + Bus + MQ + Zuul + Turbine），Alibaba 体系核心只需要一个**Nacos**+Sentinel+Gateway，部署和运维成本大幅降低。

---

## 📌 五、面试速查表

| 面试题 | 回答要点 |
|--------|---------|
| **Spring Boot 和 Spring Cloud 什么关系？** | Boot 让单个微服务跑起来，Cloud 让一群微服务协同工作。Cloud 基于 Boot，解决分布式系统的问题。 |
| **Spring Cloud 和 Spring Cloud Alibaba 什么关系？** | Cloud 是规范（定义接口），Alibaba 是实现（用 Nacos/Sentinel 替代 Eureka/Hystrix）。 |
| **Nacos 相比 Eureka 的优势？** | ①注册中心+配置中心二合一 ②AP/CP模式可切换 ③有Web控制台 ④实时配置推送 |
| **Sentinel 相比 Hystrix 的优势？** | ①多了限流能力 ②有控制台实时改规则 ③更轻量（信号量隔离） ④慢调用比例熔断 |
| **Spring Cloud Gateway 相比 Zuul？** | 非阻塞WebFlux，性能高5倍，少量线程处理大量连接 |
| **Nacos 是怎么实现配置实时推送的？** | 客户端长轮询机制，配置变更后服务端立即响应，对比 Config 需要主动 refresh |
| **微服务网关有什么作用？** | 统一入口、路由转发、鉴权、限流、熔断、跨域、协议转换 |
| **Eureka 的自我保护模式是什么？** | 网络抖动时不轻易剔除实例，宁可保留过期数据也不让服务不可用 |
