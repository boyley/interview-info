# Kafka 核心技术与实战 — 面试备战手册

> 来源：极客时间《Kafka 核心技术与实战》（胡夕，Apache Kafka 代码贡献者）
> 覆盖：消息引擎概念 → 部署配置 → 生产者/消费者原理 → 内核（副本/请求处理/重平衡）→ 运维监控 → 流处理

---

## 📌 一、面试高频考点速查

| 核心考点 | 重要度 |
|---------|-------|
| **Kafka 基本概念**（Topic/Partition/Offset/Broker/副本） | ⭐⭐⭐⭐⭐ |
| **生产端消息可靠**（acks/retries/幂等生产者/事务生产者） | ⭐⭐⭐⭐⭐ |
| **消费端可靠**（位移提交/重平衡/消费组） | ⭐⭐⭐⭐⭐ |
| **副本机制与 ISR** | ⭐⭐⭐⭐⭐ |
| **请求处理（Reactor 模型）** | ⭐⭐⭐⭐ |
| **Controller 控制器** | ⭐⭐⭐⭐ |
| **高水位与 Leader Epoch** | ⭐⭐⭐⭐ |
| **不丢消息配置** | ⭐⭐⭐⭐⭐ |
| **消费组重平衡（Rebalance）** | ⭐⭐⭐⭐ |
| **Exactly-Once 语义** | ⭐⭐⭐⭐ |
| **Kafka 为什么快**（顺序写/零拷贝/PageCache） | ⭐⭐⭐⭐⭐ |
| **Kafka Streams** | ⭐⭐⭐ |

---

## 📌 二、Kafka 核心概念

### 1. 三层架构

```
Topic 层（M 个分区 × N 个副本）
  └── Partition 层（1 个 Leader + N-1 个 Follower）
        └── Message 层（offset 从 0 开始，单调递增不可变）
```

| 概念 | 说明 |
|------|------|
| **Topic** | 消息的逻辑容器 |
| **Partition** | Topic 的分片，每个分区是**有序不可变**的消息日志，从 0 编号 |
| **Offset** | 分区内消息的位移，单调递增不可变 |
| **Broker** | 服务端进程 |
| **Leader Replica** | 承担所有客户端读写 |
| **Follower Replica** | 只从 Leader 拉取数据，**从不服务客户端**（与 MySQL 从库不同） |
| **Consumer Group** | 多个消费者共享 Group ID，每个分区同一组内只被一个实例消费 |
| **Rebalance** | 实例增减时重新分配分区（以 bug 多著称） |

### 2. 为什么 Follower 不服务读？

两个原因（面试常问）：
1. **容易实现 read-your-writes**（自读自写）——只要 Leader 读写，天然一致
2. **容易实现单调读**——不会出现读两个副本读到不同值

### 3. Kafka 不只是消息引擎

Kafka = 消息引擎 + **分布式流处理平台**（0.10.0.0 引入 Kafka Streams）。

相比外部流处理引擎（Spark/Flink），Kafka Streams 是**客户端库**而非完整平台——能提供**端到端 Exactly-Once**（外部引擎只能保证框架内 EOS，写回 Kafka 时做不到端到端）。

---

## 📌 三、部署与配置

### 1. 平台选择

| 版本 | 特点 | 适用 |
|------|------|------|
| **Apache Kafka** | 迭代最快、社区最好，缺 connector/监控 | 要掌控力 |
| **Confluent Kafka** | 创始人出品，免费版带 Schema Registry/REST | 要高级特性 |
| **CDH/HDP** | 界面化安装运维，但版本落后 | 快速搭建 |

### 2. 关键参数速查

**Broker 端**：
```
log.dirs                    = /data/kafka1,/data/kafka2   # 多磁盘目录，1.1起支持坏盘故障转移
zookeeper.connect           = zk1:2181,zk2:2181,zk3:2181/kafka1  # chroot 只加一次
auto.create.topics.enable   = false      # 防止误建 topic
unclean.leader.election.enable = false   # 防止非ISR副本成为leader导致数据丢失
auto.leader.rebalance.enable = false     # 无意义的强制leader重选
```

**JVM 端**：
```
KAFKA_HEAP_OPTS = -Xms6g -Xmx6g         # 官方推荐 6G 堆（默认1G太小）
KAFKA_JVM_PERFORMANCE_OPTS = "-server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 ..."
```

**OS 端**：
```
ulimit -n 1000000       # 避免 "Too many open files"
文件系统选 XFS（优于 ext4）
vm.swappiness ≈ 1       # 不是0（避免OOM killer，留警告时间）
```

### 3. 磁盘容量估算（面试能算）

```
例：100M 条/天 × 1KB × 2副本 = 200GB
  +10% 索引 = 220GB
  × 14 天保留 = 3TB
  × 0.75 压缩比 = 2.25TB
```

### 4. 带宽估算

```
Kafka 60%+ 的性能问题出在带宽
1Gbps 网卡，可用 70%，留 2/3 → 每 Broker ~240Mbps
目标 1TB/小时 → 2336Mb/s ÷ 240 ≈ 10 台，3副本 → 30 台
```

---

## 📌 四、生产端原理

### 1. 消息可靠投递（面试最高频）

**消息丢失的三个环节**：
| 环节 | 丢失原因 | 解决 |
|------|---------|------|
| **生产者→Broker** | `send(msg)` 是 fire-and-forget | 用 `send(msg, callback)` 检测失败 |
| **Broker 内** | Leader 崩了没复制 | `acks=all` + `replication.factor≥3`（每个分区至少 3 个副本：1 个 Leader + 2 个 Follower） |
| **消费者** | 先提交位移再消费 | 先消费后提交，手动提交 |

**不丢消息最佳实践清单**：
```
① send(msg, callback)
② acks = all                     # 所有副本确认
③ retries 设大
④ unclean.leader.election.enable = false
⑤ replication.factor >= 3          # 副本因子；每个分区至少保留 3 份副本（1 Leader + 2 Follower），可容忍最多 2 台承载该分区副本的 Broker 故障
⑥ min.insync.replicas = 2        # 最少同步副本数：acks=all 时，ISR 中至少有 2 个副本（含 Leader）确认才算写入成功；少于该值则拒绝写入，以避免只有单副本仍继续写入
⑦ replication.factor = min.insync.replicas + 1
⑧ enable.auto.commit = false     # 手动提交位移
```

### 2. 分区机制（Scalability 关键）

- 为什么分区：读写以分区为单位，加 Broker 即可水平扩展
- 默认分区策略：**带 key → hash(key)%分区数；不带 key → 轮询**
- 自定义分区器：实现 `Partitioner` 接口
- **案例**：业务标志字段自定义分区，吞吐量提升 **40 倍**

### 3. 压缩

| 算法 | 吞吐量 | 压缩比 |
|------|--------|--------|
| **LZ4** | 最高 | 中 |
| **Snappy** | 高 | 最低 |
| **zstd**（2.1+） | 中 | **最高** |
| **GZIP** | 低 | 高 |

- 压缩发生在**生产者**（`compression.type`）
- **V2 消息格式**（0.11+）：对整个消息集压缩，压缩比更好
- 注意：Broker 做消息格式转换会**破坏零拷贝**

### 4. 幂等生产者 vs 事务生产者（面试易混淆）

| 特性 | 幂等生产者 | 事务生产者 |
|------|-----------|-----------|
| **开启条件** | `enable.idempotence=true` | 幂等 + 设 `transactional.id` |
| **保证范围** | 单分区、单会话 | **跨分区、跨会话** |
| **API** | 无额外代码 | initTransactions/beginTransaction/commitTransaction |
| **消费者** | — | 需设 `isolation.level=read_committed` |
| **代价** | 低 | 高，别盲目开启 |

### 5. 生产端 TCP 连接管理

- **创建时机**：`new KafkaProducer()` 时后台 Sender 线程连接所有 bootstrap brokers；元数据更新后连所有 broker
- **关闭时机**：用户 close；`connections.max.idle.ms`（默认9分钟）；`=-1` 永久僵尸连接
- **KafkaProducer 线程安全**（通过 RecordAccumulator）

### 6. 生产端调优

**吞吐量**：`batch.size`（默认16KB太小）、`linger.ms`、开压缩（LZ4/zstd）、`buffer.memory` 调大
**延迟**：`linger.ms=0`、不压缩、避免 `acks=all`

---

## 📌 五、消费端原理

### 1. 消费组与两种消息模型

```
同一消费组内 → 队列模型（一条消息只被一个消费者消费）
不同消费组   → 发布订阅（一条消息被多个组消费）

理想实例数 = 订阅的总分区数（多了闲置，少了部分分区没人消费）
```

### 2. 位移提交

- **auto commit**：`enable.auto.commit=true` + `auto.commit.interval.ms`（默认5s）
- **手动同步** `commitSync()`：阻塞、自动重试
- **手动异步** `commitAsync()`：非阻塞、**不重试**（重试会用过期位移）
- **最佳实践**：平时异步提交，关闭前同步提交

**位移语义**：提交 offset X 表示 X 之前都消费完了。

### 3. CommitFailedException 处理

**触发**：手动 `commitSync()` 时组重平衡，分区被重新分配（`poll()` 间隔超过 `max.poll.interval.ms` 默认5分钟）。

**四种解决**：
1. 缩短每条消息处理时间
2. 调大 `max.poll.interval.ms`
3. 减少 `max.poll.records`（默认500）
4. 多线程消费

### 4. 消费端 TCP 连接管理

`new KafkaConsumer()` **不建连接**（和生产端不同），`poll()` 时才建：
1. FindCoordinator → 连最空闲 broker
2. 连接协调者 → 单独连接（隔离组协调流量和数据拉取）
3. 拉数据 → 每个持有订阅分区 Leader 的 broker 一条连接

### 5. 重平衡（Rebalance）

**触发条件**（3个）：成员数变化、订阅主题数变化、订阅主题分区数变化

**缺点**：STW（所有消费者暂停）、慢（几百成员可能要几小时）、低效（全量重分配无局部性）

**重平衡会丢消息吗？**：重平衡本身不会删除 Broker 中的消息；风险取决于 **Offset 提交时机**。

| Offset 提交时机 | 重平衡后的结果 |
|----------------|----------------|
| 先处理、后提交 | 新消费者从上一次已提交的 Offset 继续，未提交的消息会被重新处理：**可能重复，不丢消息** |
| 先提交、后处理 | 新消费者从已提交 Offset 的下一条开始，尚未处理完成的消息被跳过：**可能丢消息** |
| 自动提交 | Offset 可能已经提交而业务尚未完成，存在丢消息风险 |

**可靠做法**：关闭 `enable.auto.commit`；业务处理成功后再手动提交 Offset。发生分区撤销时，只提交已经完成处理的消息位移；下游业务做好幂等，以容忍重平衡带来的重复消费。

> 📝 **面试一句话**：Rebalance 本身不丢消息；“先提交后处理”会丢消息，“先处理后提交”会带来重复消费。

**副本未同步时又发生 Rebalance 呢？**：副本同步和消费者 Rebalance 是两套独立机制。Rebalance 只重新决定“哪个消费者消费哪个分区”，消费者仍从分区 Leader 拉取消息；因此，Follower 落后但 Leader 正常时，重平衡不会因此丢消息，未提交 Offset 的消息只会被重新消费。

真正的风险是 **落后副本尚未同步、Leader 又宕机**：

| 配置与状态 | 结果 |
|------------|------|
| `acks=all` + 合理的 `min.insync.replicas` + `unclean.leader.election.enable=false` | 只允许 ISR 副本当新 Leader；若没有可选 ISR，分区暂时不可用，优先保证已确认消息不丢 |
| `acks=1`，或 `unclean.leader.election.enable=true` | 落后的副本可能成为新 Leader；它未同步到的消息可能丢失 |

> 📝 **记忆**：Rebalance 决定“谁来消费”，副本/ISR 决定“消息是否安全”。

**避免非必要重平衡**：
```
session.timeout.ms = 6s
heartbeat.interval.ms = 2s     # 保持 session.timeout >= 3 × heartbeat
max.poll.interval.ms 调大       # 超过最大下游处理时间
修 GC 停顿（Full GC 也会触发）
```

### 6. 消费进度监控

```
方法1: kafka-consumer-groups.sh --describe --group <group>
      看 LOG-END-OFFSET / CURRENT-OFFSET / LAG

方法2: Java API AdminClient.listConsumerGroupOffsets + endOffsets

方法3: JMX records-lag-max / records-lead-min
```

---

## 📌 六、Kafka 内核

### 1. 副本机制与 ISR（面试最高频）

**ISR（In-Sync Replicas）**：与 Leader 保持同步的副本集合，Leader 永远在 ISR 中。

**同步判据不是消息数量，而是时间**：`replica.lag.time.max.ms` 是 Leader 判断 Follower 是否仍同步的最长容忍时间。Follower 在该时间内必须持续向 Leader 发起 Fetch 请求并追平 Leader 的日志末尾（LEO）；未发 Fetch 或一直未追平，Leader 就将它移出 ISR。恢复并追平后可重新加入（ISR 是动态集合）。

> 📝 **默认值与取舍**：旧版 Kafka 的默认值为 **10s**；当前 Kafka 默认 **30s**。阈值较小，故障发现更快，但短暂网络抖动也更容易造成 ISR 缩小、使 `acks=all` 写入失败；阈值较大则相反。

**Unclean Leader 选举**：选非 ISR 副本当 Leader → 可能丢数据但保可用性（CAP 取舍），`unclean.leader.election.enable=false`。

### 2. 请求处理（Reactor 模型）

```
SocketServer（Reactor 模式）
  Acceptor 线程（只分发，轮询）
    → 网络线程池（num.network.threads，默认3）
      → 共享请求队列
        → IO 线程池（num.io.threads，默认8）真正干活（写日志/读缓存）
          → 响应回各网络线程的响应队列

Purgatory：缓存延迟请求（如 acks=all 等待 ISR 确认）
```

**2.3 起数据请求和控制请求分开**处理（独立线程池/独立端口）。

### 3. Controller 控制器

- 管理整个集群：topic 增删、分区重分配、Preferred leader 选举、集群成员管理、元数据服务
- **选举**：第一个在 ZK 创建 `/controller` znode 的 broker 成为 Controller
- **单线程 + 事件队列**（0.11 重构），ZK 操作改异步（10倍提升）

### 4. 高水位（HW）与 Leader Epoch

**HW（High Watermark）**：
- 定义消息可见性（消费者只能读已提交消息，即 HW 以下）
- HW ≤ LEO（Log End Offset，下一条要写入的位移）
- 分区 HW = Leader 的 HW

**HW 的缺陷**：Follower 更新 HW 有延迟（多一轮 fetch），时间不匹配导致数据丢失/不一致。

**解决**：**Leader Epoch**（0.11+，0.11 引入）——(Epoch版本, 起始Offset)，新 Leader 查询 Leader LEO 决定截断，避免两个副本截断到旧 HW。

---

## 📌 七、Kafka 为什么快（面试加分）

| 技术 | 原理 |
|------|------|
| **顺序写** | 消息 append-only 写磁盘，顺序 IO 接近内存速度 |
| **零拷贝** | 数据从内核缓冲直接到网卡，跳过用户态拷贝（FileChannel.transferTo） |
| **PageCache** | OS 自动缓存热数据，读命中率高 |
| **Log Segment** | 日志分片，后台删除旧段 |
| **批量发送** | batch.size + linger.ms 批量传输 |

**注意**：保持客户端和 Broker 版本一致，否则消息格式转换会破坏零拷贝。

---

## 📌 八、Kafka Streams

### 1. 与其他流处理平台的区别

| 对比 | Kafka Streams | Flink/Spark |
|------|--------------|-------------|
| **形态** | Java 客户端库 | 完整平台 |
| **部署** | 用户自己管理（可嵌入微服务） | 框架管理（YARN/K8s） |
| **数据源** | 仅 Kafka | 多源 connector |
| **协调** | 消费组机制 | Master + ZK + checkpoint |
| **EOS** | **原生端到端 Exactly-Once** | 需 Kafka 0.11 事务/2PC 配合 |

### 2. 核心概念

- **KStream**：事件流；**KTable**：变更日志/状态
- **窗口**：固定窗口、滑动窗口、会话窗口（金融场景用事件时间，不是处理时间）
- **操作符**：map/filter（无状态）、groupBy/count/windowedBy（有状态）
- **案例**：实时 ID Mapping（stream-table join，leftJoin 保留未匹配记录）

---

## 📌 九、面试核心问题速查表

| 面试题 | 一句话回答 |
|--------|-----------|
| **Kafka 怎么保证不丢消息？** | 生产者 callback+acks=all；Broker replication.factor≥3+min.insync.replicas>1；消费者先消费后提交手动提交 |
| **Kafka 为什么快？** | 顺序写 + 零拷贝 + PageCache + 批量发送 |
| **Follower 为什么不服务读？** | 好实现自读自写和单调读 |
| **ISR 怎么判断同步？** | 不是消息数；Follower 必须在 `replica.lag.time.max.ms` 内持续 Fetch 并追平 Leader 的 LEO，否则移出 ISR（旧版默认10s，当前默认30s） |
| **幂等生产者和事务生产者区别？** | 幂等保证单分区单会话；事务保证跨分区跨会话 |
| **重平衡什么时候触发？** | 成员数/订阅主题数/分区数变化；有STW缺点 |
| **位移提交最佳实践？** | 平时异步 commitAsync，关闭前 commitSync |
| **Controller 怎么选的？** | 第一个在 ZK 创建 /controller znode 的 broker |
| **高水位和 LEO 区别？** | HW=已提交可见消息，LEO=下一条要写的位置，HW≤LEO |
| **Leader Epoch 解决什么？** | HW 更新延迟导致的数据丢失/不一致 |
| **Kafka Streams 相比 Flink？** | 是客户端库不是平台，原生端到端 EOS |
| **Kafka 能当分布式存储吗？** | 理论上可以（有序不可变日志） |

---

> 📝 **面试策略**：Kafka 面试先讲清**三层架构**（Topic→Partition→Message）和**副本/ISR**，这是地基。然后高频考点是**不丢消息配置**（生产者/消费者两端）+ **为什么快**（顺序写/零拷贝）。进阶加分是**Leader Epoch**（解决 HW 缺陷）、**幂等 vs 事务生产者**、**端到端 EOS**——这些能答出来基本就是 Kafka 面试的高分答案了。
