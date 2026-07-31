# 深入剖析 Java 新特性 — 面试备战手册

> 来源：极客时间《深入剖析 Java 新特性》（范学雷，Oracle 成员，JDK 5 起贡献者）
> 覆盖 JDK 9~17 核心新特性，贯穿五维评估：需求/安全/生产力/性能/维护
> 课程核心观点：JDK 8→17 合理使用新特性，可减代码~20%、减错误~20%、提升性能~20%

---

## 📌 一、面试高频考点速查

| 核心考点 | 对应 JDK | 重要度 |
|---------|---------|-------|
| **Record（档案类）** — 不可变数据载体 | JDK 16 | ⭐⭐⭐⭐⭐ |
| **Sealed Class（封闭类）** — 受限继承 | JDK 17 | ⭐⭐⭐⭐⭐ |
| **switch 表达式**（箭头标签/yield/穷尽性） | JDK 14 | ⭐⭐⭐⭐⭐ |
| **Pattern Matching（模式匹配）** | JDK 16 | ⭐⭐⭐⭐⭐ |
| **Text Block（文本块）** | JDK 15 | ⭐⭐⭐⭐ |
| **异常开销**（1000x 性能陷阱） | — | ⭐⭐⭐⭐⭐ |
| **Flow（响应式编程）** | JDK 9 | ⭐⭐⭐⭐ |
| **Vector（向量运算 SIMD）** | Incubator | ⭐⭐⭐ |
| **Foreign Memory / FFM（外部内存/函数接口）** | Incubator | ⭐⭐⭐ |
| **JShell** | JDK 9 | ⭐⭐⭐ |

---

## 📌 二、面试回答标准模板

> **面试官："Java 17 有哪些新特性？说说你最常用的几个。"**
>
> **回答**（结合课程）：
>
> "从 JDK 9 到 17 我最常用四个：**Record**、**Sealed Class**、**switch 表达式**、**模式匹配**。
>
> **Record** 用来定义不可变数据载体——一行 `record Circle(double radius)` 就自动生成构造器、equals、hashCode、toString，天然线程安全。
>
> **Sealed Class** 控制继承范围——`sealed class Shape permits Circle, Square` 限定只有这两个子类，配合 switch 模式匹配能实现**编译期变更检测**：将来新增子类，switch 会直接编译失败，逼你处理新类型。
>
> **switch 表达式**用箭头标签避免 fall-through，用 `yield` 返回值，而且要求穷尽（必须有 default），比老式 switch 安全得多。
>
> 这些特性组合起来能把错误处理性能提升几百倍，也能让代码量降 20%。"

---

## 📌 三、编码效率类特性

### 1. JShell（JDK 9）

Java 的 REPL，交互式验证小问题。语句/表达式不需要放在方法里，变量/方法不需要放在类里。

```
jshell> 1 + 1
$1 ==> 2
```

### 2. Text Block 文本块（JDK 15）

用三个双引号定义多行字符串，不需要 `\n`、`+` 拼接、`\"` 转义。

```java
String html = """
        <!DOCTYPE html>
        <html>
            <body>
                <h1>"Hello World!"</h1>
            </body>
        </html>
        """;
```

**编译期三步处理**：
1. 统一换行符为 LF（跨平台）
2. 去掉所有内容行与闭合定界符共享的前导空白 + 去掉每行尾部空白
3. 最后处理转义序列

**两个新转义**：
- `\s`（空格转义）：保留尾部空格
- `\` + 换行（续行）：跨两行连接长行

### 3. Record 档案类（JDK 16）

**定义**：不可变数据的"透明载体"。

```java
public record Circle(double radius) implements Shape {
    // compact constructor：无参无赋值，编译器自动补
    public Circle {
        if (radius < 0) throw new IllegalArgumentException("negative");
    }
    @Override public double area() { return Math.PI * radius * radius; }
}
```

**自动生成**：规范构造器、equals、hashCode、toString、访问器（`circle.radius()`）。

**Record 的约束**（保证不可变）：
- 不能 extends（隐式继承 java.lang.Record）
- final 类，不能有子类、不能 abstract
- 组件字段不可变
- 无可变实例字段、无实例初始化器
- 无 native 方法

**面试注意坑**：
- 数组组件会破坏 equals/hashCode（数组用身份比较）——`record Password(byte[] p)` 两个相同内容的数组比较为 false
- 覆盖访问器做值转换会破坏相等性

**为什么用 Record**：不可变对象天然线程安全，不用 synchronized（synchronized 方法性能差几十倍）。

### 4. Sealed Class 封闭类（JDK 17）

**解决的问题**：继承要么全开（final）要么全闭，没有中间态。Sealed 提供"可控继承"。

```java
public abstract sealed class Shape permits Circle, Square {
    public abstract double area();
}
```

**允许的子类规则**：
1. 必须在同一模块或包
2. 必须是 sealed 类的**直接**子类（permits 不传递）
3. 必须声明继续方式：`final`（关闭）/ `sealed`（继续限制）/ `non-sealed`（重新开放）

**四种扩展性限制（按优先级）**：private 类 → final → sealed → 无限制

**核心价值**：让穷尽推理成为可能。有了 sealed Shape 只允许 Circle/Square，`instanceof Square` 就能准确判断正方形——所有可能性都是已知可枚举的。

### 5. Pattern Matching for instanceof（JDK 16）

**旧的"检查+转型+绑定"三步**合并为一步：

```java
// 之前
if (shape instanceof Rectangle) {
    Rectangle rect = (Rectangle) shape;
    return rect.length == rect.width;
}
// 之后
if (shape instanceof Rectangle rect) {
    return rect.length() == rect.width();
}
```

**模式变量作用域（流式作用域）**：
- if 块内可用
- 否定 if + return 后的代码可用
- `&&` 右侧可用（短路保证先匹配）；`||` 右侧**不可用**
- 与位运算 `&` 不可用（两侧都执行）

**性能加分**：模式匹配比 cast 吞吐量高 ~20%（313M vs 263M ops/s）。

### 6. switch 表达式（JDK 14）

**老 switch 的两个问题**：break 落空（被列为软件安全漏洞）、结果变量赋值编译器无法校验。

**新 switch 表达式**：

```java
int daysInMonth = switch (month) {
    case JANUARY, MARCH, MAY, JULY, AUGUST, OCTOBER, DECEMBER -> 31;
    case APRIL, JUNE, SEPTEMBER, NOVEMBER -> 30;
    case FEBRUARY -> {
        if (leap) yield 29;
        else yield 28;
    }
    default -> throw new RuntimeException("...");
};
```

**关键规则**：
- 箭头右侧必须是**表达式、块或 throw**
- **yield** 从块中产生值，只在 switch **表达式**里合法；break 只在 switch **语句**里合法
- **穷尽性**：switch 表达式必须覆盖所有可能输入（缺 default 编译错误）——这是最大的健壮性提升

### 7. switch 模式匹配（JDK 17 preview）

switch 选择器可以是**引用类型**，case 可以是**类型模式**，直接匹配 null：

```java
public static boolean isSquare(Shape shape) {
    return switch (shape) {
        case null, Shape.Circle c -> false;
        case Shape.Square s -> true;
    };
}
```

**编译期变更检测（关键价值）**：sealed Shape 只允许 Circle/Square 时，这个 switch 是穷尽的。当基础库 v2.0 新增 Rectangle，这个 switch **停止编译**——维护者立刻知道要处理 Rectangle，而不是等用户反馈。

**性能**：switch 分发 O(1) vs if-else O(N)。

**注意**：用 default 就放弃编译期变更检测能力，确定类型不会演变才用。

---

## 📌 四、性能类特性

### 1. 异常开销：抛异常是错误处理的首选吗？（课程第08-09讲）

**实测性能**：抛异常 vs 不抛，吞吐量差 **~1000 倍**（566M vs 504K ops/s）。云付费场景下，1万元无异常的工作负载，用异常要花 1000 万。

**异常成本主要在生成栈跟踪**。

**三种异常使用模式**：
| 模式 | 适用 | 说明 |
|------|------|------|
| **可恢复异常** | catch + 忽略 + continue | 栈跟踪根本没用到，白白生成 |
| **不可恢复异常** | RuntimeException 终止 | 一次性成本，用于定位故障 |
| **记录调试信息** | 可恢复 + 记录日志 | 兼顾调用点信息和栈跟踪 |

**优化方向**：错误码 + 可开关日志——用 `new Throwable("call stack")` 记录调用栈，返回错误码；日志可开关，只在需要调试时付出成本。

**面试回答**：
> Q：Java 异常为什么慢？
> 开销主要在**生成栈跟踪**，实测抛异常比不抛慢 1000 倍。所以热路径上避免异常，用错误码/密封接口 + 记录组合替代。可恢复异常不该生成栈跟踪（反正不用），用可开关日志记录调试信息。

### 2. 错误处理的优雅方案：密封接口 + Record（第08讲）

用 sealed 接口把"返回值"和"错误码"建模成穷尽的类型：

```java
public sealed interface Returned<T> {
    record ReturnValue<T>(T returnValue) implements Returned {}
    record ErrorCode(Integer errorCode) implements Returned {}
}

// 消费者用 switch 模式匹配，两种 case 穷尽
// 编译器强制检查"用返回值前先处理错误码"
```

### 3. Flow 响应式编程（JDK 9）

**编程模型对比**：
- **命令式**："这样做"——顺序、阻塞，高并发下延迟问题凸显
- **声明式**："做什么"——回调解耦，但产生**回调地狱**
- **响应式**：数据流 + 变化传播

**Flow 四大组件**：
```
Flow.Publisher<T>   — subscribe(Subscriber)（发布者发起订阅）
Flow.Subscriber<T>  — onSubscribe / onNext / onError / onComplete
Flow.Subscription   — request(n)（背压）/ cancel()
Flow.Processor      — 既是 Subscriber 又是 Publisher（中间转换）
```

**背压（Backpressure）**：订阅者每次请求 N 个数据，控制流速。

**课程观点**：协程/虚拟线程（Project Loom）可能是响应式的后继——Go 的成功是证据。

### 4. Vector 矢量运算（Incubator，SIMD）

**目标**：SIMD——一条指令处理多个数据，利用 CPU/GPU 并行单元。

```java
FloatVector va = FloatVector.fromArray(FloatVector.SPECIES_128, a, 0);
FloatVector vy = va.mul(vx);   // 128位 = 4个float并排
```

**实测**：4维示例，标量 180M ops/s vs 向量 1.84B ops/s，**~10倍**。

**受益场景**：机器学习、线性代数、加密、金融。

### 5. Foreign Memory / FFM 外部接口（Incubator）

**外部内存接口（FMI）**解决 ByteBuffer 两个缺陷：
1. 没有资源释放接口（全靠 GC，延迟敏感库不可接受）
2. 容量是 int，上限 2G

```java
// 新 API：ResourceScope（生命周期）+ MemorySegment（连续内存区）+ MemoryAccess（读写）
try (ResourceScope scope = ResourceScope.newConfinedScope()) {
    MemorySegment segment = MemorySegment.allocateNative(4, scope);
    MemoryAccess.setByteAtOffset(segment, 0, (byte) 'A');
}
// 用 long 寻址，突破 2G 限制
```

**外部函数接口（FFM）**——纯 Java 调 C 函数，替代 JNI：

**JNI 的问题**：
1. 要 C 编译链接成平台动态库，失去"一次编写到处运行"
2. **本质上不安全**——逃出 JVM 安全模型，能访问 JDK 内部、修改不可变数据、崩溃 JVM

**FFM 优势**：全 Java 代码调 C 的 printf，不需要 C 代码和编译链接；FFM 提案希望 JNI 最终**默认禁用**。

---

## 📌 五、面试核心问题速查表

| 面试题 | 一句话回答 |
|--------|-----------|
| **Record 是什么？** | 不可变数据的透明载体，自动生成构造器/equals/hashCode/toString，天然线程安全 |
| **Sealed Class 有什么用？** | 可控继承，配合 switch 模式匹配实现编译期变更检测 |
| **switch 表达式和语句区别？** | 表达式用 yield 返回值必须穷尽；语句用 break 可 fall-through |
| **模式匹配有什么好处？** | 检查+转型+绑定一步完成，消除两类经典错误，吞吐+20% |
| **文本块怎么用？** | 三引号多行字符串，编译期三步处理（统一换行/去缩进/最后转义） |
| **Java 异常为什么慢？** | 栈跟踪生成是主要开销，实测慢1000倍，热路径避免异常 |
| **响应式编程是什么？** | 数据流+变化传播，Flow 的背压控制流速；协程可能是后继 |
| **向量运算能快多少？** | SIMD 一条指令处理多数据，实测~10倍 |
| **FFM 相比 JNI 的优势？** | 纯 Java 调 C 函数，无需编译链接，更安全，未来可能禁用 JNI |
| **JDK 8 迁移 17 有什么收益？** | 减代码~20%、减错误~20%、提升性能~20%、降维护成本~20% |

---

> 📝 **面试要点**：Java 新特性面试不用每个都讲，抓住四个核心（**Record / Sealed Class / switch 表达式 / 模式匹配**）+ 一个性能洞察（**异常开销 1000 倍**）就够了。讲的时候强调它们**组合起来**的价值——比如"密封类+模式匹配=编译期变更检测"、"Record+密封接口=优雅的错误处理"，比单讲特性更有深度。
