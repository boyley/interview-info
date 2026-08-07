# AQS 面试回答模板 — 考官问 · 你答（话术集）

> 用法：每题先记**回答结构**（3~4 步骨架），再把**话术**用自己的话复述出来，别死背。
> 配套：《Java并发与系统性能优化》给知识点（AQS 原理/锁对比/并发工具类），本文件给逐题话术。
> 面试评级：AQS 是 Java 并发必考，几乎 100% 会从"讲讲 AQS"切入，然后层层追问到源码。

---

## 🎙️ 一、万能开场与收尾

**被问"讲讲 AQS / 讲讲 ReentrantLock / 讲讲 JUC 锁"时的万能开场：**
> "AQS 是 `AbstractQueuedSynchronizer`，**整个 JUC 锁的基石**。它的核心就三样东西：一个 **`volatile int state`**（同步状态）、一个 **FIFO 的 CLH 变种等待队列**、一套基于 **CAS** 的入队与改状态操作。它用**模板方法模式**把'抢锁、排队、阻塞、唤醒'这套通用流程写死，让 ReentrantLock、Semaphore、CountDownLatch 这些锁复用同一套排队机制，子类只负责定义 `state` 的语义。"

**答任何并发题后的万能收尾（挑 1 句）：**
- "锁优化的本质，是**减少线程阻塞和唤醒的次数**。"
- "从 synchronized 到 AQS，本质是**把'没锁→加锁→尽量别阻塞'这一路演进做成了可复用的框架**。"
- "AQS 的价值不是'快'，而是把**同步状态抽象出来**，让所有锁复用同一套排队与唤醒机制。"
- "Java 层的 AQS 用 park/unpark 阻塞，JVM 层的 synchronized 走 monitor——两者是**两条技术路线**，JDK6 之后差距在缩小。"

---

## 🎙️ 二、什么是 AQS？它解决了什么问题？（必考）

**回答结构：定义 → 两个核心要素 → 模板方法模式 → 谁在用**

**话术：**
> "AQS 是 JUC 里所有锁和同步器共用的**基类**，它解决的核心问题就一个：**把'同一时刻只能有 N 个线程进入'这种同步逻辑，从每个锁各自实现、变成统一抽象**。
>
> 它抽象出两个要素：**① `state` 同步状态**——一个 volatile int，表示'当前允许进入的数量'，锁竞争就是并发地抢这个 state；**② FIFO 等待队列**——抢不到 state 的线程按顺序排成队，前一个执行完唤醒下一个。
>
> 采用的是**模板方法模式**：AQS 把 `acquire/release/acquireShared/releaseShared` 这些流程写死（抢不到就入队、排到就阻塞、释放就唤醒），子类只要实现 `tryAcquire/tryRelease/tryAcquireShared/tryReleaseShared` 几个钩子方法，**定义 state 的语义**即可。
>
> 谁在用：`ReentrantLock`（state=重入计数）、`Semaphore`（state=剩余许可）、`CountDownLatch`（state=剩余计数）、`ReentrantReadWriteLock`（state 高低 16 位分读写）、`ThreadPoolExecutor.Worker`（不可重入锁）。"

---

## 🎙️ 三、AQS 的核心原理是什么？state 和队列怎么配合？

**回答结构：state → CAS → 队列 → 三者配合的一次完整 lock**

**话术：**
> "核心是三件套：**state、CAS、CLH 队列**。
>
> **state** 是 `volatile int`，是'锁的语义'所在——它可以是重入次数、许可数、还有多少个等待者。volatile 保证多线程之间对它的修改**立刻可见**。
>
> **CAS** 是'抢锁的手段'——`compareAndSetState(0, 1)` 原子地把 state 从 0 改成 1，改成功就说明拿到锁了，**并发抢锁不用加锁**。
>
> **CLH 队列**是'抢不到时的归宿'——CAS 失败的线程通过尾节点 CAS 入队，然后阻塞等待前驱唤醒。
>
> 三者配合的完整流程：**tryAcquire 成功直接返回 → 失败 addWaiter 入队 → acquireQueued 里循环：前驱是头节点且再试一次成功就出队拿锁，否则 park 阻塞 → 持锁线程释放时 unpark 唤醒队列头部的后继**。一句话：**能抢就 CAS 抢，抢不到就排队，排队的本质是'让出 CPU + 等待唤醒'**。"

---

## 🎙️ 四、CLH 等待队列是怎么工作的？为什么是双向链表？

**回答结构：CLH 原版 vs AQS 改造 → 节点结构 → 为什么双向 → 为什么 park 代替纯自旋**

**话术：**
> "原始 CLH 是**单向链表 + 纯自旋**：每个线程等待的是前驱节点的状态，靠自旋轮询前驱。AQS 对 CLH 做了改造，变成**双向链表 + 阻塞唤醒**。
>
> 每个节点存 `thread` 和 `waitStatus`：`CANCELLED`(1) 取消、`SIGNAL`(-1) 表示后继需要被唤醒、`CONDITION`(-2) 在条件等待队列、`PROPAGATE`(-3) 共享模式传播唤醒。
>
> 为什么是**双向**？两个原因：**① 入队后要回头找前驱节点，把前驱的 waitStatus 设置成 SIGNAL**，这样前驱释放锁时才记得唤醒我；**② 线程被中断或超时取消时，要能方便地把自己从队列里摘掉**。
>
> 为什么用 **park/unpark 阻塞**而不是纯自旋？纯自旋在锁竞争激烈时**白白烧 CPU**；阻塞把 CPU 让出去，唤醒再回来。AQS 的取舍是：**入队和拿锁之间短自旋尝试，失败就 park 阻塞**，把自旋的省心留给了 CAS 抢锁的那一下。"

---

## 🎙️ 五、独占模式和共享模式有什么区别？

**回答结构：判定标准 → 钩子方法不同 → 唤醒方式不同 → 各自代表**

**话术：**
> "区别在于**'同时能有几个线程成功'**。
>
> **独占模式**：同一时刻只有一个线程能拿到——`tryAcquire` 返回 boolean，true 拿到、false 排队。代表：`ReentrantLock`。
> **共享模式**：多个线程可以同时拿到——`tryAcquireShared` 返回**剩余可用数**：负数=失败入队，>=0 表示还能放这么多线程进来。代表：`Semaphore`、`CountDownLatch`、读锁。
>
> 最关键的差异在**唤醒**：独占释放 `release` 只唤醒**头节点的下一个**；共享释放 `releaseShared` 会通过 `doReleaseShared` **传播式唤醒多个**——因为一个线程释放后，可能还有多个线程在等同一个共享资源，一次唤醒一个会白白把后面的人又阻塞一轮。"
> 
> 补充一句：`ReentrantReadWriteLock` 最妙——**读锁是共享模式、写锁是独占模式**，靠 state 高 16 位存读锁计数、低 16 位存写锁计数，一个 int 同时表达两把锁。

---

## 🎙️ 六、ReentrantLock 和 synchronized 有什么区别？

**回答结构：四个高级能力 → 实现路线不同 → 性能演进 → 怎么选**

**话术：**
> "ReentrantLock 比 synchronized 多了**四个高级能力**：
> **① 可中断**——`lockInterruptibly()` 在等待锁时能被 `interrupt()` 打断；**② 可超时**——`tryLock(timeout)` 等不到就返回，不傻等；**③ 公平锁**——可以按申请顺序发锁；**④ 多个 Condition**——一个锁能配多个条件队列，而 synchronized 只有 wait/notify 一个条件。
>
> 实现路线不同：**synchronized 是 JVM 层面的**，基于对象头的 monitor（偏向锁→轻量级锁→重量级锁的升级）；**ReentrantLock 是 Java 层面的**，基于 AQS 的 state + 队列。
>
> 性能演进：JDK6 给 synchronized 加了锁升级，两者性能**差距已经很小**，甚至无竞争时 synchronized 更省。
>
> 怎么选：**简单同步用 synchronized**（不用手动释放、不会漏锁），**需要可中断/超时/公平/多条件这些高级能力时用 ReentrantLock**。一句话：synchronized 是'够用就好'，ReentrantLock 是'需要什么给什么'。"

---

## 🎙️ 七、公平锁和非公平锁怎么实现的？公平锁真的公平吗？

**回答结构：实现差异 → 默认选谁/为什么 → "公平"的真相**

**话术：**
> "实现差异就在**入队之前那一小步**：
>
> **非公平锁**（ReentrantLock 默认）：线程进来**先 `compareAndSetState(0,1)` 直接抢一次**，抢到了就直接进临界区，**根本不去看队列**——这就是'插队'。抢失败才走标准的 acquire 排队。
> **公平锁**：acquire 之前先调 **`hasQueuedPredecessors()`**，判断等待队列里**是不是已经有线程在排队**了，有就乖乖排到队尾，不抢。
>
> 为什么默认非公平？因为**减少线程切换开销、吞吐更高**——被唤醒的线程从 park 到真正拿到锁要重新调度，这期间新来的线程直接 CAS 抢，往往已经执行完了，锁'空转'的窗口被利用上。代价是队列里的线程可能被长期饿着，但概率低、可接受。
>
> 公平锁真的公平吗？**不是严格公平**。它只保证'新来的线程不插队'，但拿锁的顺序受 CPU 调度、park 唤醒时机影响，只能说**基本按 FIFO**。所以文档里叫'公平性尽可能高'，别答成'绝对公平'。"

---

## 🎙️ 八、可重入是怎么实现的？

**回答结构：owner + state 计数 → 获取/释放对称 → synchronized 也有**

**话术：**
> "可重入的本质是**'同一个线程可以反复拿同一把锁，锁的持有是带计数的'**。
>
> ReentrantLock 里维护一个 **`owner` 线程 + `state` 计数**：线程进来时如果 `owner == currentThread`，**`state+1` 直接重入成功**，不用再走队列；线程退出时 `state-1`，**减到 0 才真正释放锁**并唤醒后继。
>
> 所以 `lock()` 三次就要 `unlock()` 三次，**获取和释放必须对称**，少一次锁就永远不释放。
>
> 补充：**synchronized 也是可重入的**，基于 monitor 的重入计数。这点是死记硬背也一定要会的——'可重入'是两套锁共有的性质，区别只是实现位置不同。"

---

## 🎙️ 九、画一下 lock() 的完整源码流程？

**回答结构：两步走（先抢再排）→ 入队 → 循环阻塞 → 对称的释放**

**话术：**
> "以**非公平 ReentrantLock.lock()** 为例，完整流程：
>
> ```
> lock():
>   ① compareAndSetState(0, 1)        // 进来先抢一次，非公平锁的插队
>       成功 → 设 owner = 当前线程，直接拿锁，结束
>   ② acquire(1):                     // 抢失败才进来
>      a. tryAcquire(1)               //   子类实现，再试一次（可重入判断也在这）
>          成功 → 返回
>      b. addWaiter(Node.EXCLUSIVE)   //   CAS 入队尾，构造等待节点
>      c. acquireQueued(node):        //   核心循环
>          前驱是 head 且 tryAcquire 成功
>            → setHead(node) 出队，返回
>          否则 shouldParkAfterFailedAcquire  // 把前驱设为 SIGNAL
>            → parkAndCheckInterrupt         // park 阻塞，等 unpark
> ```
>
> 释放对称：`unlock()` → `release(1)` → `tryRelease(1)`（state 减到 0 才算成功）→ **`unparkSuccessor(head)`** 唤醒队列里 head 的下一个节点，被唤醒的线程回到 acquireQueued 循环里再试。
>
> 面试要点：**获取是'CAS 抢 + 排队 + 阻塞'三步，释放是'减计数 + 唤醒后继'两步**，把这条线画出来，再往下聊信号量、CountDownLatch 都是同一套骨架。"

---

## 🎙️ 十、Condition 的原理？await/signal 做了什么？条件队列和同步队列什么关系？

**回答结构：两条队列 → await/signal 各做什么 → 为什么这么设计 → 怎么用**

**话术：**
> "AQS 里其实有**两条队列**：**同步队列**（抢锁失败者的等待区，管互斥）和**条件队列**（持锁者主动让位、等条件满足的等待区，管条件）。两者用同一个 Node 类，靠 `waitStatus == CONDITION` 标识当前在条件队列。
>
> **`await()` 做三件事**：① 构造节点（waitStatus=CONDITION）**加入条件等待队列**；② **释放当前持有的锁**（`fullyRelease`，把可重入计数 state 整个释放并保存）；③ `park` 阻塞。
>
> **`signal()` 做两件事**：① 把条件队列**队首节点转移回 AQS 同步队列**（`enq()`）；② **不立即唤醒**——节点回到同步队列公平排队，真正被 unpark 要等锁释放后由前驱唤醒、重新抢到锁。`signalAll` 就是全转移。
>
> 为什么这样设计：**语义必须分离**——一把锁既要管互斥、又要管'条件满足'；多个 Condition 才能**精确唤醒**（有界队列的'不满/不空'两个条件），这正是管程的条件变量机制。
>
> 和 Object.wait/notify 对比：**一个锁配多个 Condition**、支持**超时 await(nanos)** 和**可中断 awaitInterruptibly**，更精确。"

### ① 两条队列对比（追问常考）

| 维度 | AQS 同步队列 | Condition 条件等待队列 |
|------|-------------|----------------------|
| 谁在排队 | 抢锁失败的线程 | 已持锁、await 主动让位的线程 |
| 管什么 | 互斥（谁能拿锁） | 条件（条件满足没有） |
| 被谁唤醒 | 前驱释放锁 → unpark | 条件满足 → signal() |
| 结构 | 双向链表（prev/next） | 单向链表（nextWaiter） |
| 数量 | 每把锁一条 | 一个 Condition 一条，可多条 |
| 节点状态 | SIGNAL / CANCELLED | CONDITION(-2) |

### ② 怎么用（生产者-消费者，三条铁律）

```java
Lock lock = new ReentrantLock();
Condition notFull  = lock.newCondition();  // 生产者等"不满"
Condition notEmpty = lock.newCondition();  // 消费者等"不空"

lock.lock();
try {
    while (queue.isFull()) {  // ① while 防虚假唤醒
        notFull.await();      // ② 条件不满足：让出锁，睡觉
    }
    enqueue(item);
    notEmpty.signal();        // ③ 唤醒一个消费者
} finally {
    lock.unlock();
}
```

- **铁律一**：await/signal 必须在 `lock()/unlock()` 之间（持锁调用，否则抛 IllegalMonitorStateException）
- **铁律二**：await 用 `while` 不用 `if`——**防虚假唤醒**，且 await 返回后必须**重新检查条件**
- **铁律三**：signal 只是把节点挪回同步队列排队，**真正唤醒要等它重新抢到锁**

### ③ await/signal 完整时序

```
生产线程持锁，发现队列满
  → notFull.await()
      ├─ 节点(CONDITION) 加入 notFull 条件队列
      ├─ 释放锁（保存可重入计数 savedState）
      └─ park 阻塞
消费线程持锁，取走元素
  → notEmpty.signal()
      ├─ notFull 队首节点 → enq() 转回同步队列
      └─ 节点与其他抢锁线程公平排队
消费线程 unlock() → unparkSuccessor 唤醒它
  → 生产线程抢到锁 → await 返回 → while 再检查队列是否还满
```

> 📌 一句话记忆：**同步队列管"谁有资格进"，条件队列管"条件有没有满足"；signal 不是叫醒，是把睡着的节点送回抢锁队伍。** `ArrayBlockingQueue` 底层就是 `ReentrantLock + notEmpty/notFull 两条 Condition`。

---

## 🎙️ 十一、AQS 支持线程中断吗？怎么实现？

**回答结构：两种获取方式 → 中断处理差异 → 面试坑点**

**话术：**
> "支持，但要分清**两种获取方式对中断的态度不同**：
>
> **`lock()`（不可中断）**：线程即使被 interrupt，也**继续在队列里排队等锁**，拿到锁之后才把中断标志补上（`selfInterrupt`）。也就是说**它不响应中断，只记录中断**。
> **`lockInterruptibly()`（可中断）**：底层走 `acquireInterruptibly`，在**入队前和 park 返回后**都会检查中断标志，一旦发现被中断，**直接抛 `InterruptedException` 出队**，放弃获取。
>
> 面试坑点：别答成'AQS 支持中断'就完了——要能说出**默认的 lock 是忽略中断的，只有 lockInterruptibly/tryLock 才响应**；synchronized 想中断只能等锁释放或直接不响应，这是 ReentrantLock 的一个核心卖点。"

---

## 🎙️ 十二、AQS 在 JUC 里具体被哪些类用？state 分别代表什么？

**回答结构：一张表打天下 → 说一个最特殊的（Worker/读写锁）**

**话术：**
> "AQS 的 state 是**一个 int，语义由子类定义**，一张表说清：

| 同步器 | 模式 | state 的语义 |
|--------|------|-------------|
| `ReentrantLock` | 独占 | 重入计数（0=空闲） |
| `ReentrantReadWriteLock` | 读共享/写独占 | 高 16 位读锁计数，低 16 位写锁计数 |
| `Semaphore` | 共享 | 剩余许可数 |
| `CountDownLatch` | 共享 | 还需等待的计数 |
| `ThreadPoolExecutor.Worker` | 独占 | 是否被占用（0/1），实现不可重入独占 |

> 最值得讲的两个：**读锁/写锁共用一个 int**——用位运算把 state 劈成两半，一把 AQS 同时表达两把锁；**Worker**——不直接用 ReentrantLock，而是**自己继承 AQS 实现一把不可重入的锁**，保证 runWorker 执行任务期间 worker 不被并发接管。讲出这两个，考官就知道你是真的看过源码而不是背八股。"

---

## ✅ 使用检查清单

- [ ] 能 30 秒讲清"AQS = state + CLH 队列 + CAS + 模板方法"
- [ ] 能画出 lock()/unlock() 的完整源码流程图（先抢→入队→阻塞→唤醒）
- [ ] 能讲清独占 vs 共享、公平 vs 非公平、可重入的实现差异
- [ ] 能说清 CLH 为什么改双向、为什么 park 而不是纯自旋
- [ ] 能讲透 await/signal 与 wait/notify 的区别、多 Condition 的价值
- [ ] 能背出 state 语义表，并讲出读写锁位拆分、Worker 不可重入锁这两个"看过源码"的信号
- [ ] 每次回答收尾都用一句金句（减少阻塞唤醒次数 / 两条技术路线）
