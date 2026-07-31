# 透视 HTTP 协议 — 面试备战手册

> 来源：极客时间《透视 HTTP 协议》（罗剑锋，十余年网络协议分析与 Nginx 魔改经验）
> 涵盖 42 讲：HTTP 演进 → 报文结构 → 连接管理 → 缓存/Cookie → 代理 → HTTPS/TLS → HTTP/2/HTTP/3 → WebSocket → 性能优化

---

## 📌 一、面试高频考点速查

| 核心考点 | 重要度 |
|---------|-------|
| **HTTP 报文结构**（起始行+头部+空行+body） | ⭐⭐⭐⭐⭐ |
| **八种请求方法**（安全/幂等/语义） | ⭐⭐⭐⭐⭐ |
| **状态码分类**（1xx-5xx + 301/302/304/206 详解） | ⭐⭐⭐⭐⭐ |
| **HTTP 特性**（无状态/明文/队头阻塞） | ⭐⭐⭐⭐⭐ |
| **HTTPS/TLS 握手**（对称+非对称混合/ECDHE 流程） | ⭐⭐⭐⭐⭐ |
| **TLS 1.3**（1-RTT/0-RTT/前向安全） | ⭐⭐⭐⭐⭐ |
| **HTTP/2 多路复用 + HPACK** | ⭐⭐⭐⭐⭐ |
| **HTTP/3 / QUIC**（为什么用 UDP） | ⭐⭐⭐⭐ |
| **HTTP 缓存机制**（Cache-Control/ETag/304） | ⭐⭐⭐⭐⭐ |
| **Cookie 机制**（属性/SameSite/HttpOnly） | ⭐⭐⭐⭐ |
| **连接管理**（长连接/队头阻塞/并发连接） | ⭐⭐⭐⭐ |
| **WebSocket**（握手/全双工/帧结构） | ⭐⭐⭐ |
| **CDN 原理**（GSLB/DNS/回源） | ⭐⭐⭐ |
| **Nginx/OpenResty**（epoll/进程池） | ⭐⭐⭐ |

---

## 📌 二、HTTP 演进史

```
HTTP/0.9（1991） 只允许 GET，纯文本，响应后关连接
HTTP/1.0（1996） 增加 HEAD/POST、状态码、Header；参考文档非正式
HTTP/1.1（1999） ★正式标准：PUT/DELETE、持久连接、chunked、强制 Host、缓存控制
HTTP/2  （2015） 源于 Google SPDY；二进制、多路复用、HPACK、服务器推送
HTTP/3  （2018） 基于 QUIC（UDP）
```

**面试回答**：
> Q：HTTP 版本演进最关键的变化是什么？
> 1.0 到 1.1 的关键是**持久连接**（长连接）和强制 **Host 头**（实现虚拟主机）。1.1 到 2 的关键是**多路复用**（一个连接并发多个请求，解决队头阻塞）。2 到 3 是把底层从 TCP 换成 **QUIC**（UDP），解决 TCP 层的队头阻塞。

---

## 📌 三、HTTP 报文

### 1. 报文结构

```
HTTP 报文 = 起始行 + 头部字段集合 + 空行(CRLF) + 消息正文

请求行：GET / HTTP/1.1      ← 方法 + URI + 版本
状态行：HTTP/1.1 200 OK     ← 版本 + 状态码 + 原因

头部字段规则：
  字段名不区分大小写、不能有空格、用连字符不能用下划线
  ':'前不能有空格，值前可有多个空格
  Host 是唯一必须出现的请求字段（虚拟主机区分）
```

### 2. 八种请求方法

| 方法 | 语义 | 安全 | 幂等 |
|------|------|------|------|
| **GET** | 获取资源 | ✅ | ✅ |
| **HEAD** | 只返回响应头 | ✅ | ✅ |
| **POST** | 新建/提交（INSERT） | ❌ | ❌ |
| **PUT** | 修改（UPDATE） | ❌ | ✅ |
| **DELETE** | 删除 | ❌ | ✅ |
| CONNECT | 建隧道 | ❌ | ❌ |
| OPTIONS | 列出可实行方法 | ❌ | ✅ |
| TRACE | 追踪路径（有漏洞，禁用） | ❌ | ✅ |

### 3. 状态码分类

| 分类 | 含义 | 关键码 |
|------|------|--------|
| **1xx** | 提示 | 101 Switching Protocols（WebSocket） |
| **2xx** | 成功 | 200 OK / 204 No Content / **206 Partial Content**（范围请求） |
| **3xx** | 重定向 | **301** 永久 / **302** 临时 / **304** 缓存验证通过 |
| **4xx** | 客户端错误 | 400 / 403 / 404 / 405 / 429 Too Many Requests |
| **5xx** | 服务器错误 | 500 / 502 / 503（配合 Retry-After） |

---

## 📌 四、HTTP 特性

| 特性 | 说明 | 优点/缺点 |
|------|------|----------|
| **灵活可扩展** | 只规定基本格式，可任意加头字段/方法 | ✅ 万能协议 |
| **可靠传输** | 基于 TCP，尽量保证（约3-4个9） | ✅ 不是100% |
| **请求-应答** | 请求方主动，应答方被动 | ✅ 契合 C/S |
| **无状态** | 每次报文独立，无记忆 | ✅ 易集群 ❌ 无法多步事务→Cookie |
| **明文** | 肉眼可读 | ✅ 易调试 ❌ 易窃听 |
| **不安全** | 无身份认证、无完整性校验 | ❌ → 引出 HTTPS |

**队头阻塞**：请求-应答串行队列，队首慢连累后续。
**缓解**：并发连接（浏览器 6~8 个）、域名分片。**终极方案**：HTTP/2 多路复用。

---

## 📌 五、连接管理

```
短连接：3 次握手 1 RTT + 4 次挥手 2 RTT = 3 RTT，请求应答才 2 RTT → 浪费 60%
长连接（HTTP/1.1 默认）：Connection: keep-alive，成本均摊
关闭：客户端 Connection: close；服务器 keepalive_timeout / keepalive_requests
```

**传输大文件四种方法**：
- **压缩**：gzip/br（对图片无效）
- **分块 chunked**：`Transfer-Encoding: chunked`，每块=长度+数据
- **范围请求 Range**：`Range: bytes=x-y` → 206 + Content-Range（视频拖拽/断点续传）
- **多段数据**：multipart/byteranges

---

## 📌 六、Cookie 与缓存

### Cookie（给无状态 HTTP 加记忆）

```
响应 Set-Cookie: key=value → 浏览器保存 → 请求 Cookie: k1=v1; k2=v2

属性：
  生存期：Expires（绝对时间）/ Max-Age（相对秒，优先）
  作用域：Domain / Path
  安全：HttpOnly（防XSS）/ SameSite（防CSRF）/ Secure（仅HTTPS）
```

### HTTP 缓存（面试最高频）

```
服务器控制：
  Cache-Control: max-age=30     ← 新鲜度
  no-store（不许缓存）/ no-cache（用前验证）/ must-revalidate（过期必须验证）

客户端控制：
  Cache-Control: max-age=0（要最新）/ no-cache（Ctrl+F5 强制刷新）

条件请求：
  If-Modified-Since ↔ Last-modified（时间）
  If-None-Match ↔ ETag（实体标签，解决时间精度问题）
  → 未变返回 304，浏览器复用缓存

口诀："没有请求的请求，才是最快的请求。"
```

**代理缓存新属性**：private/public、s-maxage（代理存活）、no-transform（禁止改写数据）

---

## 📌 七、HTTPS / TLS

### 1. HTTPS = HTTP over SSL/TLS

**安全四性**：机密性（加密）、完整性（不被篡改）、身份认证（证明身份）、不可否认。

### 2. 加密体系

| 类型 | 算法 | 用途 |
|------|------|------|
| **对称加密** | AES、ChaCha20 | 加密数据（快，13MB/s） |
| **非对称** | RSA、ECC | 密钥交换（慢，15KB/s） |
| **摘要** | SHA-2、HMAC | 完整性校验 |
| **签名** | 私钥加密摘要 | 身份认证 |

**混合加密**：开始用非对称交换会话密钥，之后全部用对称加密。

### 3. TLS 1.2 ECDHE 握手流程

```
① Client Hello（随机数、密码套件列表）
② Server Hello（随机数、选定套件）
③ Server Certificate（证书）
④ Server Key Exchange（椭圆曲线公钥+签名）
⑤ Server Hello Done
⑥ 客户端验证证书链 + 签名
⑦ Client Key Exchange（客户端公钥）
⑧ 双方 ECDHE 算出 Pre-Master
⑨ 三个随机数 → Master Secret（48字节）→ 派生会话密钥
⑩ Change Cipher Spec + Finished
```

### 4. TLS 1.3 特性

| 改进 | 说明 |
|------|------|
| **1-RTT 握手** | 删除 Key Exchange 消息，Client Hello 直接带公钥 |
| **0-RTT** | pre_shared_key + early_data，有重放攻击风险 |
| **前向安全** | 废除 RSA/DH，只用 ECDHE——私钥泄露不影响历史会话 |
| **HKDF** | 替代 PRF |
| **兼容** | 伪装成 TLS1.2（记录头 Version 固定 0x0303） |

### 5. HTTPS 优化

- **TLS1.3**（1-RTT）+ ECDHE + x25519 曲线
- **会话复用**：Session ID / Session Ticket
- **OCSP Stapling**：服务器预取 OCSP 响应随证书发
- **HSTS**：`Strict-Transport-Security`，浏览器自动转 HTTPS
- **SNI**：Client Hello 带域名，实现 HTTPS 虚拟主机

---

## 📌 八、HTTP/2

### 1. 核心特性

| 特性 | 说明 |
|------|------|
| **二进制帧** | 9 字节报头（长度+类型+标志+流ID） |
| **多路复用** | 一个连接多个流并发，解决应用层队头阻塞 |
| **HPACK 头部压缩** | 静态表+动态表+哈夫曼，压缩率 50%~90% |
| **伪头字段** | :authority/:method/:status（替代起始行） |
| **服务器推送** | Server Push |
| **强化安全** | 事实上强制加密，要求 TLS1.2+、前向安全 |

### 2. 帧结构（9 字节报头）

```
0               1               2               3
0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
+---------------+---------------+---------------+---------------+
|         帧长度(3字节)        |  帧类型(1字节) | 标志位(1字节) |
+---------------+---------------+---------------+---------------+
|                  流标识符（4字节，31位可用）                 |
+--------------------------------------------------------------+
```

**流（Stream）**：帧的双向传输序列，流 ID 不能重用只能递增（客户端奇数、服务器偶数），0 号流只发控制帧。

**连接前言**：客户端发 `PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n` 24 字节。

**ALPN**：TLS 扩展协商应用层协议（h2 > http/1.1）。

### 3. 面试回答

> Q：HTTP/2 怎么解决队头阻塞？
> HTTP/2 把消息拆成二进制**帧**，多个帧通过不同**流（Stream）**在一个 TCP 连接上**多路复用**。队首请求慢不再阻塞后续请求。但 TCP 层仍有队头阻塞——丢包时所有流都等重传。这正是 HTTP/3 用 QUIC（UDP）解决的核心问题。

---

## 📌 九、HTTP/3 / QUIC

| 特性 | 说明 |
|------|------|
| **基于 UDP** | 无连接、无需握手，在 UDP 上实现可靠传输 |
| **解决 TCP 队头阻塞** | 单个流阻塞，其他流不受影响 |
| **内含 TLS1.3** | 0-RTT/1-RTT |
| **连接 ID** | 不绑定 IP+端口 → **连接迁移**（4G↔WiFi 不中断） |
| **双向/单向流** | 流 ID 最低 2 位标志 |
| **QPACK 头部压缩** | 静态表 98 个，解决 HPACK 队头阻塞 |

**服务发现**：`Alt-Svc: h3=host:port` 扩展帧告知 HTTP/3 端点。

---

## 📌 十、WebSocket

**定义**：基于 TCP 的轻量级全双工通信协议（RFC 6455），与 HTTP 平级。

**为什么需要**：HTTP 请求-应答是半双工+服务器被动，难做实时通信；浏览器沙盒不能用 TCP。

**特点**：全双工（服务器主动推送）、二进制帧、协议名 ws/wss、端口 80/443 穿透防火墙。

**握手（协议升级）**：
```
请求：GET / + Connection: Upgrade + Upgrade: websocket
      + Sec-WebSocket-Key（Base64 的 16 字节随机数）
响应：101 Switching Protocols
      + Sec-WebSocket-Accept = base64(sha1(key + 固定UUID))
```

**帧结构**：FIN + Opcode（1文本/2二进制/8关闭）+ MASK（客户端必须掩码）+ Payload len（变长）。

---

## 📌 十一、Nginx / CDN

### Nginx 高性能原理

```
进程池 + 单线程：master 管 worker，worker 绑定 CPU，消除线程切换
I/O 多路复用 epoll：连接可读可写才处理，几十万并发仅几百M内存
多阶段处理：handler → filter → upstream → balance，11 个阶段流水线
```

### OpenResty

- Nginx + ngx_lua（内嵌 Lua），一站式 Web 平台
- **代码热加载**：从磁盘/Redis 加载，不停机更新配置
- **同步非阻塞**：基于 Lua 协程，比 epoll 更省资源
- **LuaJIT**：JIT 编译成本地机器码，接近 C 速度

### CDN 原理

```
就近访问 + 缓存代理：
GSLB（全局负载均衡）→ DNS 返回 CNAME → 选最合适边缘节点
缓存系统：命中（返回缓存）/ 回源（取源站）
指标：命中率（商业 90%+）/ 回源率
```

---

## 📌 十二、面试核心问题速查表

| 面试题 | 一句话回答 |
|--------|-----------|
| **HTTP 报文结构？** | 起始行+头部+空行(CRLF)+body |
| **GET 和 POST 区别？** | 语义（获取vs提交）、安全（GET安全）、幂等（GET幂等） |
| **301 和 302 区别？** | 301永久（换域名）、302临时（系统维护），都配合 Location |
| **HTTP 为什么无状态？** | 每次报文独立不记忆；易集群但多步事务要用 Cookie |
| **队头阻塞是什么？** | 串行队列队首慢连累后续；HTTP/2 多路复用解决应用层，QUIC 解决 TCP 层 |
| **缓存机制？** | Cache-Control 新鲜度 + ETag/Last-Modified 条件请求 + 304 |
| **TLS 握手过程？** | 非对称换会话密钥，之后对称加密；ECDHE 前向安全 |
| **TLS1.3 改进？** | 1-RTT/0-RTT、前向安全、HKDF |
| **HTTP/2 多路复用？** | 二进制帧+流，一个连接并发多个请求 |
| **HTTP/3 为什么用 UDP？** | QUIC 在 UDP 上实现可靠+多路复用，解决 TCP 队头阻塞 |
| **WebSocket 和 HTTP 区别？** | 全双工、服务器可主动推送，握手走 Upgrade+101 |
| **HTTPS 怎么优化？** | TLS1.3、会话复用、OCSP Stapling、HSTS、CDN 卸载 |

---

> 📝 **面试策略**：HTTP 是面试基础必考。核心掌握**报文结构、状态码、缓存、Cookie、HTTPS/TLS、HTTP/2**。讲 HTTP/2 时用"多路复用解决应用层队头阻塞，但 TCP 层还在；HTTP/3 用 QUIC 彻底解决"这个递进逻辑，能体现理解深度。
