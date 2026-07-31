# 深入拆解 Tomcat & Jetty — 面试备战手册

> 来源：极客时间《深入拆解 Tomcat & Jetty》（李号双）
> 涵盖 42 讲：必备基础 → 整体架构 → 连接器 → 容器 → 通用组件 → 性能优化
> 核心认知：几乎所有的 Java Web 框架（如 Spring）都是对 Servlet 的封装；Tomcat = "HTTP 服务器 + Servlet 容器"

---

## 📌 一、面试高频考点速查

| 核心考点 | 重要度 |
|---------|-------|
| **Tomcat 整体架构**（Server→Service→Connector+Container） | ⭐⭐⭐⭐⭐ |
| **连接器三层结构**（EndPoint/Processor/Adapter） | ⭐⭐⭐⭐⭐ |
| **容器层级**（Engine→Host→Context→Wrapper）+ Mapper 定位 | ⭐⭐⭐⭐⭐ |
| **Pipeline-Valve 责任链** | ⭐⭐⭐⭐⭐ |
| **生命周期管理**（LifeCycle 接口/一键式启停） | ⭐⭐⭐⭐ |
| **NioEndpoint 原理**（LimitLatch/Acceptor/Poller） | ⭐⭐⭐⭐⭐ |
| **类加载器打破双亲委派**（WebAppClassLoader） | ⭐⭐⭐⭐⭐ |
| **Web 应用隔离**（Common/Catalina/Shared/WebApp 类加载器） | ⭐⭐⭐⭐ |
| **热加载 vs 热部署** | ⭐⭐⭐⭐ |
| **Session 管理** | ⭐⭐⭐⭐ |
| **线程池调优**（利特尔法则/I-O 公式） | ⭐⭐⭐⭐⭐ |
| **Tomcat vs Jetty 对比** | ⭐⭐⭐⭐ |
| **零拷贝/APR** | ⭐⭐⭐⭐ |
| **Spring Boot 内嵌 Tomcat** | ⭐⭐⭐ |

---

## 📌 二、必备基础

### 1. Web 容器是什么

```
Tomcat/Jetty = "HTTP 服务器 + Servlet 容器"

Servlet 没有 main 方法，必须由容器实例化并调用
Spring 应用本身就是一个 Servlet
JBoss/WebLogic 是完整 Java EE 服务器（含 EJB 容器）；Tomcat/Jetty 是轻量级
Spring Boot 默认内嵌 Tomcat
```

### 2. Servlet 接口与生命周期

```java
public interface Servlet {
    void init(ServletConfig config);     // 容器加载时调用（如 Spring MVC DispatcherServlet 在此建容器）
    ServletConfig getServletConfig();
    void service(ServletRequest req, ServletResponse res);  // ★ 最重要
    String getServletInfo();
    void destroy();                       // 卸载时调用
}
```

**请求流程**：HTTP 服务器封装 ServletRequest → 容器 service → 根据 URL 映射找 Servlet（未加载则反射创建+init）→ 调 service → 返回响应。

**扩展机制**：
- **Filter**（过滤器）：基于过程行为，链接成 FilterChain，`doFilter` 调下一个
- **Listener**（监听器）：基于状态，监听 Web 应用启停/请求到达事件

### 3. Cookie 与 Session

```
Cookie：HTTP 报文的一个请求头，本质是存储在用户本地的文件
Session：服务器端存储空间，通过 Cookie 携带 Session ID 关联请求
Java 中 Session 由容器调用 getSession() 时创建
集群部署需 Session 共享（Redis），后台线程定期清理过期 Session
```

---

## 📌 三、Tomcat 整体架构

### 1. 顶层结构

```
Server（Tomcat 实例）
  └── Service（只做组装）
        ├── Connector（多个，负责对外交流）
        │     └── HTTP/1.1、AJP、HTTP/2 协议 + NIO/NIO2/APR I/O 模型
        └── Container（一个 Engine，负责内部处理）
```

**连接器（Connector）**：负责 Socket 连接、字节流 ↔ Request/Response 对象转化。
**容器（Container）**：加载管理 Servlet、处理 Request。

### 2. 连接器三层结构（面试核心）

```
Connector
  ├── EndPoint   ← 传输层抽象（TCP/IP）：监听/接收/发送 Socket
  ├── Processor  ← 应用层协议抽象（HTTP/AJP）：字节流解析成 Request
  └── Adapter    ← 适配器模式：Tomcat Request → 标准 ServletRequest → 调容器 service
```

**ProtocolHandler 接口**：封装"通信协议 + I/O 模型"两个变化点（如 `Http11NioProtocol`、`AjpNioProtocol`）。

**设计思路**：找变化点/不变点，接口+抽象基类封装不变点，抽象基类定义模板方法，子类实现变化点。

### 3. 容器层级与 Mapper 定位

```
Engine（引擎，管理多个虚拟站点）
  └── Host（虚拟主机/站点，按域名）
        └── Context（一个 Web 应用，按路径）
              └── Wrapper（一个 Servlet）
```

**Mapper 组件**：将请求 URL 定位到 Servlet。一个 URL 最终只定位到一个 Wrapper：

```
URL: http://user.shopping.com:8080/order/buy
① 根据端口选 Service 和 Engine
② 根据域名 user.shopping.com 选 Host
③ 根据路径 /order 找 Context
④ 根据 web.xml Servlet 映射路径找 Wrapper
```

### 4. Pipeline-Valve 责任链

```java
public interface Valve {
    void invoke(Request request, Response response);  // 内部调 getNext().invoke() 触发下一个
}
```

- Pipeline 维护 Valve 链表，**BasicValve** 在链表末端，负责调下层容器的第一个 Valve
- 入口：`connector.getService().getContainer().getPipeline().getFirst().invoke(request, response)`
- **Valve vs Filter**：Valve 是 Tomcat 私有机制，工作于容器级别，拦截所有应用请求；Filter 是公有 Servlet API，工作于应用级别，只拦某个 Web 应用

### 5. 生命周期管理（一键式启停）

**LifeCycle 接口**：init/start/stop/destroy。父组件 init 中调子组件 init（组合模式），只需启动顶层 Server。

**LifeCycleBase 抽象基类**（模板方法模式）：实现公共逻辑（状态维护、事件触发、监听器管理），子类实现 `initInternal/startInternal/stopInternal/destroyInternal`。

```
init() 模板方法四步：
状态检查（必须 NEW）→ 触发 INITIALIZING 事件 → 调 initInternal() → 触发 INITIALIZED 事件
```

**Service.startInternal 启动顺序**：先 Engine → 再 MapperListener → 最后连接器（内层先启动才能对外服务）。

**启动流程**：startup.sh → Bootstrap（初始化类加载器）→ Catalina（解析 server.xml 创建组件）→ Server.start → Service → Connector+Engine。

**关闭钩子**：CatalinaShutdownHook 在 JVM 停止前执行 Server.stop；Server.await() 监听 8005 端口收 "SHUTDOWN" 命令。

---

## 📌 四、连接器：三种 I/O 模型

### 1. I/O 模型对比

| 模型 | 特点 | Tomcat 实现 |
|------|------|------------|
| **BIO** | 一连接一线程，阻塞 | 已废弃 |
| **NIO** | 同步非阻塞 + Selector 多路复用 | `Http11NioProtocol`（默认） |
| **NIO.2** | 异步回调 | `Http11Nio2Protocol`（Windows 更好） |
| **APR** | JNI 调 C 库 | `Http11AprProtocol`（TLS 场景高性能） |

### 2. NioEndpoint 五大组件

```
NioEndpoint
  ├── LimitLatch     ← 连接数控制器（默认10000，基于 AQS）
  ├── Acceptor       ← 单线程阻塞 accept，把 SocketChannel 封装成 PollerEvent 入队
  ├── Poller         ← Selector，检测 I/O 事件，可读则生成 SocketProcessor
  ├── SocketProcessor ← Runnable，调 Http11Processor 解析请求
  └── Executor       ← Tomcat 定制线程池执行
```

**高并发思路**：把接收连接（Acceptor）、检测 I/O 事件（Poller）、处理请求（Executor）三件事分开，用不同规模的线程组处理。

### 3. APR 提速两大秘密（面试加分）

**① DirectByteBuffer 避免内存拷贝**：
```
HeapByteBuffer：内核→临时本地内存→JVM 堆（多一次拷贝，GC 可能移动对象）
DirectByteBuffer：内核→本地内存（少一次拷贝，快好几倍）
AprEndpoint 用 DirectByteBuffer；Nio/Nio2 用 HeapByteBuffer（稳定性考虑）
```

**② sendfile 零拷贝**：
```
传统：磁盘→内核→本地→堆→本地→内核→网卡（6次拷贝+切换）
sendfile：文件读入内核缓冲，只把位置/长度描述符加到 Socket 缓冲，数据直接内核→网卡
```

### 4. Tomcat 定制线程池

**与原生 ThreadPoolExecutor 的区别**：原生在总线程数达 maximumPoolSize 时立即拒绝；Tomcat 捕获 RejectedExecutionException 后**继续尝试把任务放入队列**，队列也满才拒绝。

**定制版 TaskQueue（重写 offer）**：线程数已达 max → 入队；有空闲线程 → 入队；poolSize < max → 返回 false 触发创建新线程。

---

## 📌 五、容器：类加载与 Web 应用隔离

### 1. 类加载器体系（打破双亲委派）

```
CommonClassLoader
  ├── CatalinaClassLoader  （Tomcat 自身类）
  └── SharedClassLoader    （Web 应用共享类，如 Spring）
        └── WebAppClassLoader（每个 Web 应用一个）
```

**WebAppClassLoader 打破双亲委派**：优先加载 Web 应用自己的类（findClass 先在本地目录找），但用 ExtClassLoader 加载 JRE 核心类（防止自定义 Object 覆盖核心类）。

**三个隔离目标**：
- 同名 Servlet 隔离 → 每个 WebApp 一个加载器
- 共享第三方 JAR → SharedClassLoader
- Tomcat 自身类隔离 → CatalinaClassLoader（与 WebApp 平行）

### 2. 线程上下文类加载器（TCCL）

**问题**：Spring 由 SharedClassLoader 加载，但业务 Bean 在 Web 应用目录，Spring 的类加载器找不到。

**解决**：Tomcat 启动 Web 应用时 `Thread.currentThread().setContextClassLoader(webAppClassLoader)`，Spring 用 `getContextClassLoader()` 加载 Bean。JDBC 也用 TCCL 加载不同数据库驱动。

### 3. 热加载 vs 热部署

| | 热加载 | 热部署 |
|--|-------|--------|
| **范围** | 重新加载类 | 重新加载整个 Web 应用 |
| **Session** | 不清空 | 清空 |
| **场景** | 开发环境 | 生产环境 |

- Tomcat 用 `ScheduledThreadPoolExecutor` 开启后台线程（`ContainerBackgroundProcessor`），只需在顶层 Engine 启动一个，递归执行所有子容器的周期任务
- 热加载核心：WebappLoader 调 Context.reload()——销毁重建 Context 和类加载器，**保留 Session**
- 热部署由 Host 通过监听器 HostConfig 实现（监听 PERIODIC_EVENT 检查目录/WAR）

### 4. Servlet/Filter/异步管理

**Servlet 延迟加载**：默认启动时不创建 Servlet 实例（除非 loadOnStartup=true）。

**Filter 链**：ApplicationFilterChain 内部维护 Filter 数组 + pos 指针，靠 Filter 自身调 chain.doFilter 触发下一个，最后一个调 Servlet.service。

**异步 Servlet**（Servlet 3.0）：`startAsync()` 拿 AsyncContext，Web 应用用单独线程处理耗时操作，Tomcat 线程立即返回线程池。超时默认 30 秒。

---

## 📌 六、Session 管理与集群

### 1. Session 管理

- 每个 Context 一个 Manager，默认 StandardManager
- `getSession(true)` → Request → Context → Manager.createSession（加入 `ConcurrentHashMap<String, Session>`）
- 清理：后台线程每 10s backgroundProcess，取模 6 → 每 60s 检查过期 Session
- **外观类（Facade）**：开发者拿到的是 RequestFacade/StandardSessionFacade，防止暴露内部细节

### 2. 集群通信（组播）

```
每个节点周期性（500ms）发组播心跳包，3s 未收到则视为崩溃

两种同步方式：
  DeltaManager（默认）：all-to-all 拷贝，节点<4 时合适
  BackupManager：只拷到一个备份节点，节点多时更高效
```

---

## 📌 七、性能优化

### 1. 线程池大小（面试核心公式）

**利特尔法则**：
```
线程池大小 = 每秒请求数 × 平均请求处理时间
```

**I/O 时间与 CPU 时间公式**：
```
线程池大小 = (I/O 阻塞时间 + CPU 时间) / CPU 时间    ← I/O 密集需更多线程
```

**实践**：先用公式估算，再压测迭代——逐步加大直到 TPS 不再升或下降即为最佳。

### 2. 连接数配置

```
Tomcat 最大并发连接数 = maxConnections + acceptCount
  maxConnections：任意时刻处理的最大连接数（NIO 默认 10000）
  acceptCount：accept 队列长度（默认 100，对应 backlog）
  队列满 → 内核发 RST → 客户端 Connection reset

调优：net.core.somaxconn（Linux 默认128）和 acceptCount 一起调大
```

### 3. 内存溢出排查（实战步骤）

```
① -verbose:gc -Xloggc:gc.log -XX:+PrintGCDetails 开 GC 日志
② jstat -gc <pid> 2000 1000 看各代使用率
③ GCViewer 分析：年老代持续上升+频繁 Full GC = 泄漏特征
④ jmap -dump:live,format=b,file=heap.bin <pid> 堆转储
⑤ Eclipse MAT 分析 Heap Dump 定位泄漏对象
```

**GC 调优思路**：年轻代高位频繁 Minor GC → 调大年轻代；年老代高位 Full GC → 看 Full GC 后是否降下来（降→调大年老代；不降→怀疑泄漏）。

### 4. CPU 过高排查（实战步骤）

```
① top 看进程 CPU
② top -H -p <pid> 看线程 CPU
③ jstack <pid> 定位线程栈（找死循环）
④ 若无单线程高 CPU → 怀疑上下文切换 → vmstat 看 cs（上下文切换）in（中断）
```

### 5. 网络优化（sysctl）

```
net.core.somaxconn = 4096        # accept 队列
net.core.netdev_max_backlog = 16384
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_recycle = 1      # TIME_WAIT 复用
文件句柄：ulimit -n 40000
```

---

## 📌 八、Tomcat vs Jetty（面试高频对比）

| 对比项 | Tomcat | Jetty |
|--------|--------|-------|
| **架构** | 固定多级容器骨架（Engine/Host/Context/Wrapper） | Handler 责任链（松散易扩展） |
| **扩展** | 自定义 Valve（学习成本较高） | HandlerCollection 构建链、ScopedHandler 控制顺序 |
| **资源消耗** | 较重 | 更轻量（可去掉 SessionHandler 省内存） |
| **线程池** | 每个连接器自己的线程池 | 所有连接器共享一个全局线程池 |
| **稳定性** | 成熟稳定，企业级 | 偶有错误率 |
| **适用** | 关键企业级应用 | 嵌入式/资源受限/快速开发 |
| **内置使用** | Spring Boot 默认 | Hadoop、Solr 内嵌 |

**性能实测**（同一 Spring Boot 应用）：Jetty 吞吐略高、线程和内存消耗明显更少；但 Tomcat 无错误、Jetty 有 2.45% 错误率。

**一句话**：Tomcat 是成熟稳重的工程师（不会轻易出错），Jetty 是灵活可塑的后起之秀（快但偶尔犯错）。

---

## 📌 九、面试核心问题速查表

| 面试题 | 一句话回答 |
|--------|-----------|
| **Tomcat 连接器三层结构？** | EndPoint（传输层）+ Processor（协议层）+ Adapter（适配到 ServletRequest） |
| **容器层级？** | Engine→Host→Context→Wrapper，Mapper 定位 URL 到 Servlet |
| **Pipeline-Valve 和 Filter 区别？** | Valve 是 Tomcat 私有、容器级、拦所有请求；Filter 是 Servlet API、应用级 |
| **如何一键式启停？** | LifeCycle 接口，父组件组合启动子组件，只需启动顶层 Server |
| **NioEndpoint 组件？** | LimitLatch（连接数）+ Acceptor（接收）+ Poller（Selector）+ SocketProcessor + Executor |
| **为什么打破双亲委派？** | WebAppClassLoader 优先加载应用自己的类，实现应用间类隔离 |
| **TCCL 解决什么？** | Spring 由 SharedClassLoader 加载但 Bean 在应用目录，用线程上下文加载器 |
| **热加载和热部署区别？** | 热加载重载类保留Session（开发）；热部署重载应用清Session（生产） |
| **线程池大小怎么算？** | 利特尔法则：QPS×处理时间；IO密集公式：(IO+CPU)/CPU |
| **APR 为什么快？** | DirectByteBuffer 少拷贝 + sendfile 零拷贝 |
| **Tomcat vs Jetty？** | Tomcat 稳定成熟重；Jetty 轻量灵活快 |
| **Connection reset 怎么排查？** | 检查 acceptCount + net.core.somaxconn 是否太小 |

---

> 📝 **课程精华**：学 Tomcat 不只是背架构，而是学**组件化设计思想**——面向接口、抽象基类模板方法、责任链、观察者（生命周期事件）、对象池、类加载隔离。面试时讲"Tomcat 怎么打破双亲委派"、"NioEndpoint 怎么拆分三件事"、"线程池怎么算"，都能体现对开源中间件的源码级理解。
