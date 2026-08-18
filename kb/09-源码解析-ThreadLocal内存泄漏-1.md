# 09 源码解析 · ThreadLocal 内存泄漏（机器层深挖）

> **前置知识**：建议先读 [09-并发编程.md](09-并发编程.md) 的 `7.1 ThreadLocal 原理与内存泄漏（线程池串号踩坑）`（结论层），本篇扒 **OpenJDK 8（Corretto 8.452）源码**：ThreadLocal 为什么"每个线程一份"、ThreadLocalMap 的弱引用 Entry 是怎么设计的、内存泄漏到底漏在哪一环、线程池串号的完整链路、为什么"用完必须 remove"。
>
> 实测环境（本篇所有探针数字）：**Corretto 8.452 / Windows 11 / i7-12700 / 16G**；探针源码在 `_probe/TLProbe.java`。

## 目录（纯文本，锚点待软件实测后启用）

1. 一、ThreadLocal 原理（1.1–1.2）
2. 二、源码拆解：set / get / Entry（2.1–2.5）
3. 三、内存泄漏（3.1–3.3）
4. 四、线程池串号（4.1–4.2）
5. 五、InheritableThreadLocal 与实战清单（5.1–5.3）
6. 六、异步场景：上下文怎么跨线程传递（6.1）
6. 收尾：核心方法对照表 + 关键设计思想 + 面试 Q&A

---

## 一、ThreadLocal 原理

### 1.1 是什么：每个线程一份"私人储物柜"

> **说明**：`ThreadLocal` 提供**线程局部变量**——同一个 ThreadLocal 对象，在不同线程里 get 到的是各自的值，互不干扰。它**不是"存在 ThreadLocal 对象里"，而是存在"当前线程自己的 Map 里"**（`Thread.threadLocals`），ThreadLocal 对象只是"钥匙/下标"。

生活类比：**公司每人一个私人储物柜**——钥匙（ThreadLocal 对象）是公用的（大家手里都是同一把钥匙的样子），但柜子（存储）是每人自己的（线程私有）。你放进柜子里的东西，别人拿同一把钥匙去开自己的柜子，开出来的还是他自己的东西。

```java
// 基本用法（JDK 8）：
ThreadLocal<String> tl = new ThreadLocal<>();
tl.set("线程A的值");        // 写进"当前线程"的 Map
String v = tl.get();        // 从"当前线程"的 Map 读
tl.remove();                // 清除（见第四章：为什么必须 remove）
```

> **结论**：**ThreadLocal 的本体是 `Thread.threadLocals`（每个线程一个 ThreadLocalMap），ThreadLocal 实例只是 key**。这一句理解到位，泄漏、串号、InheritableThreadLocal 全都能推出来。

### 1.2 ThreadLocalMap：线程私有的"特殊 HashMap"

> **说明**：`ThreadLocalMap` 是 ThreadLocal 的内部类，**每个 Thread 有一个**（`Thread.threadLocals` 字段），惰性创建。它**不是 HashMap**，而是一个**自定义的开放地址法哈希表**（线性探测解决冲突，不是链地址法）：

```java
// ThreadLocal.ThreadLocalMap（JDK 8，核心字段）：
static class ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal<?>> {   // ★ 弱引用 key！
        Object value;                                            // value 是强引用
        Entry(ThreadLocal<?> k, Object v) { super(k); value = v; }
    }
    private Entry[] table;          // 开放地址：线性探测，不是链表
    private int size = 0;
    private int threshold;          // 扩容阈值 = len * 2/3（见 3.3）
    // 初始容量 16，扩容时翻倍
}
```

> **注意（与 HashMap 的三点不同）**：① **key 是弱引用**（WeakReference）——这是防泄漏的第一道设计；② **冲突用线性探测**（向后找空位，而不是链表/红黑树）；③ **扩容阈值是 2/3**（不是 HashMap 的 3/4）。这三个细节决定了它的回收与清理策略（见第三章）。

---

## 二、源码拆解：set / get / Entry

### 2.1 set / get：绕不开"当前线程"这个主角

> **说明**：ThreadLocal 的 set/get 超级简单——**先拿当前线程，再拿它的 Map，然后操作**（JDK 8）：

```java
// ThreadLocal（JDK 8，set/get 全貌）：
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);        // Thread.threadLocals
    if (map != null) map.set(this, value);
    else createMap(t, value);              // 首次 set 才建 Map
}
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);   // 用"自己"当 key 找
        if (e != null) return (T) e.value;
    }
    return setInitialValue();              // 没 set 过 → 返回初始值(null)并登记
}
ThreadLocalMap getMap(Thread t) { return t.threadLocals; }   // 就在 Thread 上
static ThreadLocalMap createMap(Thread t, T firstValue) {
    return new ThreadLocalMap(this, firstValue);
}
```

> **结论**：**ThreadLocal 本身几乎不存数据**——`set` 把 (this, value) 放进当前线程的 Map，`get` 拿 this 去当前线程的 Map 里找。**"线程隔离"的真相 = 数据存在各自线程的 Map 里，天然隔离**。

### 2.2 getEntry：弱引用 key 的查找路径

> **说明**：查找同样走开放地址（JDK 8）：

```java
// ThreadLocalMap.getEntry（JDK 8）：
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);  // ① 哈希定位（2^n 取模）
    Entry e = table[i];
    if (e != null && e.get() == key) return e;             // ② 直接命中（快路径）
    return getEntryAfterMiss(key, i, e);                   // ③ 冲突 → 线性探测
}
// getEntryAfterMiss 探测过程中：遇到 key==null 的陈旧 Entry，
// 顺手调用 expungeStaleEntry(i) 清理 —— 这是"读路径顺手打扫"的设计（见 3.3）
```

> **注意**：`e.get()` 是 WeakReference 的方法——**返回弱引用指向的对象；如果 ThreadLocal 已被 GC，e.get() 返回 null**。所以"key==null 的 Entry"是内存泄漏的直接标记。

### 2.3 Entry 为什么是弱引用：防"key 泄漏"的第一道闸

> **说明**：关键设计——**Entry 的 key（ThreadLocal 对象）用弱引用**。这意味着：**只要外部不再强引用这个 ThreadLocal 对象（变量置 null/方法结束），GC 就能回收 key**，Entry 变成"key=null 的陈旧条目"：

```java
// 引用强度对比（JDK 8 ThreadLocalMap.Entry 的语义）：
ThreadLocal<byte[]> tl = new ThreadLocal<>();
tl.set(new byte[1024 * 1024]);     // 1MB 数据
tl = null;                         // ① 外部强引用断了
// ② GC 后：key（ThreadLocal 对象）是弱引用 → 被回收 → Entry.get()==null
// ③ 但 value（1MB byte[]）仍是强引用，被 Entry → table → threadLocals → Thread 强持有！
//    → value 泄漏（直到线程死亡或 Map 清理触发）
```

> **结论**：**弱引用只解决了"key 的泄漏"，没解决"value 的泄漏"**——value 是强引用，只要线程活着、Map 不清理，value 就一直在。这正是"ThreadLocal 内存泄漏"这个经典问题的病灶所在。

### 2.4 哈希魔数 0x61c88647：为什么 ThreadLocal 的散列不撞车

> **说明**：ThreadLocal 的"钥匙编号"不是普通 `hashCode()`，而是一个**每次递增固定魔数的序列**（JDK 8）：

```java
// ThreadLocal（JDK 8，哈希生成）：
private final int threadLocalHashCode = nextHashCode();   // 每个实例拿一个
private static AtomicInteger nextHashCode = new AtomicInteger();
private static final int HASH_INCREMENT = 0x61c88647;     // ★ 魔数 = 1640531527

private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);        // 每次 + 0x61c88647
}
```

> **说明（数字推导）**：`0x61c88647` 是**黄金分割比例在 32 位整数域的近似**——`2^32 × (1 − 1/φ) ≈ 2654435769 × 0.618 ≈ 1640531527`（φ 为黄金比）。它保证：**在 2^n 的 table 容量下，任意两个 ThreadLocal 的 `hash & (len-1)` 分布高度均匀**（斐波那契散列），把线性探测的冲突压到最低。

> **注意（为什么不用普通 hashCode）**：普通 `Object.hashCode()` 在同一个 JVM 内可能连续、聚集（尤其在对象数量少时），直接取模会扎堆，线性探测下冲突链变长。**递增魔数 + 2^n 容量 = 伪随机序列 + 快速取模**——这正是"小而精"的哈希设计：不求通用，只求当前场景（每线程少量 key）下不撞车。

### 2.5 开放地址 vs 链地址：为什么 ThreadLocalMap 不用 HashMap 的链表法

> **说明**：HashMap 用**链地址法**（冲突挂链表/红黑树），ThreadLocalMap 用**开放地址法**（冲突线性向后找空位）——同一道题两种解法，各有取舍：

| 维度 | HashMap（链地址） | ThreadLocalMap（开放地址） |
|---|---|---|
| 冲突解决 | 链表/红黑树 | 线性探测（向后找空位） |
| 负载因子 | 0.75 | **2/3**（更低） |
| 内存布局 | 节点对象散落（不连续） | **数组连续（缓存友好）** |
| 数据规模 | 可能极大（百万级） | **每线程几个~几十个（极小）** |
| 删除 | 摘链表节点 | **置 null + 重排探测链**（expungeStaleEntry） |
| 弱引用清理 | 无此需求 | 顺带清扫陈旧 Entry（见 3.3） |

> **结论**：**开放地址在"小规模 + 连续数组 + 频繁清理"场景更优**——ThreadLocal 每线程就几个 key，数组全在缓存里（探测几次就命中），且删除/清理逻辑（置 null + 重排）在数组上简单直接；HashMap 的链表法更适合大容量、通用场景。**"规模决定数据结构"是选型的第一原则**——面试常问"为什么 ThreadLocalMap 不用 HashMap 的实现"，答案就在这张表。

---

## 三、内存泄漏

### 3.1 泄漏链：Thread → Map → Entry → value 的强引用

> **说明**：完整泄漏链一张图钉死（**价值 1MB 的 byte[] 为什么回收不掉**）：

```mermaid
flowchart LR
    T[线程 Thread<br/>存活（如线程池常驻线程）] -->|"强引用"| M[ThreadLocalMap]
    M -->|"强引用 table[]"| E[Entry]
    E -->|"强引用 value"| V[1MB byte[] 值<br/>泄漏点！]
    K[ThreadLocal 对象<br/>已置 null] -.->|"弱引用<br/>GC 可回收"| E
    style V fill:#ffcccc,stroke:#c00
```

> **说明**：**key（弱引用）回收了，但 value（强引用）还挂在这条链上**——只要线程活着（线程池的线程常驻不销毁），这 1MB 就永远占用。**这正是"ThreadLocal 泄漏"与"普通对象泄漏"的区别：它不是忘了清引用，而是弱 key + 强 value 的结构性残留**。

### 3.2 泄漏实证：GC 后 key 没了，value 还在

> **说明**：本机实测（`TLProbe.weakKey`，反射看 ThreadLocalMap 内部）：

```java
// TLProbe.java —— 置 null + GC 后，检查 Map 里的陈旧 Entry（节选）：
ThreadLocal<byte[]> tl = new ThreadLocal<>();
tl.set(new byte[1024 * 1024]);     // 1MB payload
tl = null;                          // 断外部强引用
System.gc(); Thread.sleep(500);
System.gc(); Thread.sleep(500);
// 反射数出"key==null 的 Entry 个数"（key 已被 GC，value 还在）
```

```
// 实测输出（Corretto 8.452）：
[weak] set 1MB payload; entry count=3
[weak] after tl=null + GC: null-key entries=1   ← key 被回收（弱引用生效）
// 但 1MB 的 value 仍然被 Map 强持有 —— 直到线程死亡或下一次 set/get 触发清理
```

> **注意（数字的意义）**：entry count=3（主线程 Map 里 3 个条目：探针 1MB + 前面实验的残留 + JVM 内部使用）；GC 后 null-key=1（**弱引用 key 确实被回收了**）——但**这 1MB value 还占着**。要验证"value 没被回收"，最直接的方式：开 `-Xmx64m` 反复 set 1MB 大对象而不 remove，观察 OOM（经典演示，本篇用引用链分析代替）。

### 3.3 为什么不自动清理：懒清理策略与 2/3 阈值

> **说明**：既然有陈旧 Entry，为什么 GC 不顺手清掉 value？**因为"清掉 value"需要知道"这个 key 再也不会用了"——GC 管不到业务语义**。ThreadLocalMap 的清理是**懒的**：只在 **set / get / remove 的路径上**顺带清理（JDK 8）：

```java
// ThreadLocalMap 的清理触发点（JDK 8 语义）：
// ① getEntryAfterMiss：探测时遇到 key==null → expungeStaleEntry（顺路扫）
// ② set：插入前先探测清理（cleanSomeSlots + 必要时 replaceStaleEntry）
// ③ remove：精确删除目标 Entry 并顺带清理一段
// ④ 扩容（size >= threshold = len*2/3）前先全面清理（expungeStaleEntries）

// expungeStaleEntry：把陈旧的 Entry 置 null（value 也置 null！），并重哈希后续冲突项
private int expungeStaleEntry(int staleSlot) {
    table[staleSlot].value = null;     // ★ value 置 null —— 这才是真正释放 1MB 的地方
    table[staleSlot] = null;
    // ... 线性探测重排后续 Entry，保证 get 的探测链不断
}
```

> **结论**：**"用完 set/get/remove 触发清理"是防泄漏的第二道闸，但它不可依赖**——如果线程 set 之后再也不碰这个 ThreadLocal（比如一次性的短任务），**没有任何清理会被触发，value 就永久残留**。所以唯一的根治是：**用完主动 `remove()`**（见 4.2）。2/3 阈值的意思：`size >= len*2/3` 就扩容，保证线性探测的效率（探测链不会太长）。

---

## 四、线程池串号

### 4.1 串号实证：上一个任务的 ThreadLocal 值被下一个任务读到

> **说明**：**线程池场景是 ThreadLocal 泄漏 + 串号的重灾区**——因为池里的线程是复用的、常驻的。任务 A set 的值，任务 B 用同一个线程执行时 get 到的是**任务 A 的残留值**——这就是"串号"（串数据）。本机实测（`TLProbe.poolBleed`）：

```java
// TLProbe.java —— 单线程池：task1 set，task2 get（节选）：
ThreadLocal<String> tl = new ThreadLocal<>();
ExecutorService pool = Executors.newFixedThreadPool(1);
pool.submit(() -> { tl.set("user-1001"); });          // task1：写
Thread.sleep(200);
pool.submit(() -> System.out.println("task2 reads TL=" + tl.get()));  // task2：读
```

```
// 实测输出（Corretto 8.452）：
[bleed] task1 sets TL=user-1001
[bleed] task2 reads TL=user-1001   ← 串号！task2 读到 task1 的用户
// 业务后果：用户在 task2 里看到了上一个请求的用户身份 —— 数据越权/串数据
```

> **注意（为什么必然串号）**：**线程池线程不销毁** → `Thread.threadLocals` 不清空 → 下一个任务复用同一线程 → get 命中残留值。**这不是偶发，是确定性的**——只要 task1 没 remove，task2 必然读到。**ThreadLocal 在池化环境下的正确姿势只有一句：用完必须 remove**。

### 4.2 根治：remove 的三个时机

> **说明**：`remove()` 精确删除当前线程 Map 里这个 key 的条目（含 value 置空），**是防泄漏与防串号的唯一根治手段**（JDK 8）：

```java
// 正确姿势（三个时机，任选其一覆盖"用完"语义）：
// ① finally 兜底（最稳）：
ThreadLocal<Object> tl = new ThreadLocal<>();
try {
    tl.set(expensiveValue());
    // ... 业务逻辑
} finally {
    tl.remove();            // ★ 无论成功失败，用完即清
}

// ② 线程池任务里（配合 try-finally，同上）；
// ③ 框架层拦截器/切面统一清理（AOP：请求结束 remove 该请求用到的所有 ThreadLocal）
```

```java
// ThreadLocal.remove（JDK 8）：
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null) m.remove(this);      // 精确删除：value 置 null + 顺带清理
}
```

> **结论（防串号口诀）**：**"set 一次，必配 remove 一次"**——尤其线程池、请求级上下文（用户信息、TraceId、事务上下文）这类"值有归属"的 ThreadLocal。**Static 的 ThreadLocal（全局共享 key）+ 池化线程 + 不 remove = 生产事故三件套**。

---

## 五、InheritableThreadLocal 与实战清单

### 5.1 InheritableThreadLocal：父子线程传递的坑

> **说明**：`InheritableThreadLocal` 让**子线程创建时继承父线程的值**（JDK 8 的 `Thread.init` 里复制 `inheritableThreadLocals`）：

```java
// InheritableThreadLocal（JDK 8）：
// 父线程 set 的值，new Thread(...) 创建子线程时自动复制一份到子线程
InheritableThreadLocal<String> itl = new InheritableThreadLocal<>();
itl.set("parent-value");
new Thread(() -> System.out.println(itl.get())).start();   // 子线程能读到 parent-value
```

> **注意（两个坑）**：① **只复制一次**——子线程创建之后，父线程再改值，子线程看不到（复制是创建瞬间的快照）；② **线程池场景更坑**——池里线程是"别人的子线程"，继承的是**创建它那次**的值，且复用后**值会串**（和 4.1 同款串号）。**生产上用"传递上下文"优先选显式传参或 ThreadLocal + 任务包装器**，别依赖 InheritableThreadLocal 的隐式魔法。

### 5.2 典型应用：请求上下文 / TraceId / 线程安全工具类

> **说明**：ThreadLocal 的四大生产场景（全部围绕"**线程私有一份**"这个特性）：

| 场景 | 做法 | 为什么用 ThreadLocal |
|---|---|---|
| **请求上下文**（用户/租户/权限） | 拦截器 set，业务代码 get | 贯穿一次请求的多个方法，免去层层传参 |
| **TraceId 日志链路** | 入口生成 TraceId 塞 ThreadLocal，日志模板读取 | 一次请求的所有日志带上同一 TraceId，跨方法/跨库排查 |
| **线程安全工具类实例**（SimpleDateFormat/随机数） | 每线程一个实例，避免共享实例的竞态 | 实例非线程安全，但"每线程各一份"就天然隔离 |
| **事务/连接上下文**（Spring/MyBatis） | 框架把 Connection/SqlSession 放 ThreadLocal | 一个事务横跨多个 DAO 调用，共享同一连接 |

```java
// 典型：TraceId 贯穿（简化版）：
public class TraceContext {
    private static final ThreadLocal<String> TRACE = new ThreadLocal<>();
    public static void set(String traceId) { TRACE.set(traceId); }   // 入口过滤器设置
    public static String get() { return TRACE.get(); }
    public static void clear() { TRACE.remove(); }                   // 请求结束清理
}
// 日志模板：%X{traceId} 或自定义 pattern 读 TraceContext.get()
```

> **注意（配套纪律）**：① 请求级上下文**必须在请求结束（finally/过滤器）remove**，否则下个请求串号（见第四章）；② **跨线程传递上下文不要裸用 ThreadLocal**——异步任务（线程池/CompletableFuture）要显式把上下文传进去（或包装器在任务内 set、finally remove）；③ **别用 ThreadLocal 代替方法参数**传业务数据——隐式依赖会让代码难以追踪。

### 5.3 实战清单：ThreadLocal 的 10 条纪律

> **说明**：把前面所有源码层结论落成检查清单（对照自检）：

```
[ ] 1. ThreadLocal 声明为 private static final（防多实例 key 错乱、防类加载重复）
[ ] 2. set 后必须配套 remove（finally 里），尤其线程池/请求上下文
[ ] 3. 值对象尽量小 —— 大对象（如 1MB byte[]）泄漏代价高
[ ] 4. 不用 InheritableThreadLocal 做跨线程传递（快照语义 + 池化串号）
[ ] 5. 不用 ThreadLocal 存"可变共享对象"（如 SimpleDateFormat 线程安全问题要用
[ ]     线程池局部实例，不是把共享实例塞 ThreadLocal 就完事）
[ ] 6. 排查泄漏：jmap -histo 看 ThreadLocalMap$Entry 数量 / MAT 查引用链
[ ] 7. 框架（Spring Security/MyBatis 等）的 ThreadLocal 由框架清理，别手动 remove
[ ]     别人的 key（可能清错）；自己 set 的自己清
[ ] 8. 别用 ThreadLocal 当"全局变量"传参 —— 隐式依赖难排查
[ ] 9. 任务包装器统一清理：ExecutorService 提交前包一层 finally remove（AOP 兜底）
[ ] 10. 生产监控：线程池任务完成后检查 threadLocals 是否残留（可埋点统计）
```

> **收尾一句话**：ThreadLocal 的机器层 = "每个线程一个开放地址哈希表 + 弱引用 key + 强引用 value"；它用弱引用解决了 key 的泄漏，却把 value 的清理留给开发者——**"用完 remove"不是洁癖，是这个数据结构的契约**。

---

## 六、异步场景：上下文怎么跨线程传递

### 6.1 任务包装器与 TTL：线程池/异步下的 ThreadLocal 传递

> **说明**：线程池和异步编排（CompletableFuture）场景，**子任务跑在别的线程上，ThreadLocal 天然丢失**（存储在线程自己的 Map 里）。三种解法按复杂度递增：

```java
// 解法1：任务包装器（不引依赖，最朴素）——进入任务时 set，结束 finally remove
public class TraceRunnable implements Runnable {
    private final Runnable task;
    private final String traceId;                    // 提交时从调用线程捕获
    public TraceRunnable(Runnable task) {
        this.task = task;
        this.traceId = TraceContext.get();           // 捕获调用线程的上下文
    }
    public void run() {
        TraceContext.set(traceId);                   // 子线程里恢复
        try { task.run(); }
        finally { TraceContext.remove(); }           // 用完清，防串号
    }
}
// 提交：pool.execute(new TraceRunnable(task));     // 包装后提交

// 解法2：CompletableFuture 场景——supplyAsync 前先捕获，任务内手动恢复（同上思路）
// 解法3：TTL（TransmittableThreadLocal，阿里开源）——线程池复用下自动传递/回传，
//        适配 execute/submit/supplyAsync；代价是引入依赖 + 侵入线程池（TtlExecutors.getTtlExecutor）
```

> **注意（为什么不用 InheritableThreadLocal 解决）**：它只在"创建子线程瞬间"复制一次快照，**线程池的线程早就创建好了**（不是任务提交时创建的），所以继承机制对线程池**完全失效**——这正是"线程池 + ThreadLocal"问题不能靠 InheritableThreadLocal 解决的根源，只能靠**"任务级包装"（提交时捕获、执行时恢复、结束清理）**。

> **结论（选型）**：**少量场景手写包装器即可；全链路 TraceId/上下文传递用 TTL 一步到位**；原则不变——**谁把值带进线程，谁负责清掉**。

---

## 收尾：核心方法对照表 + 关键设计思想 + 面试 Q&A

### 核心方法对照表

| 方法 | 作用 | 关键实现 |
|---|---|---|
| `ThreadLocal.set/get` | 存/取线程局部值 | 拿当前线程的 Map 操作 |
| `ThreadLocal.remove` | 精确清除 | Map.remove → value 置 null |
| `ThreadLocalMap` | 线程私有哈希表 | 开放地址 + 线性探测 |
| `Entry` | 弱 key + 强 value | `extends WeakReference<ThreadLocal>` |
| 哈希魔数 | `0x61c88647` 递增 | 黄金分割散列，2^n 容量下均匀 |
| `getEntryAfterMiss` | 探测 + 顺带清理 | expungeStaleEntry |
| `expungeStaleEntry` | 清理陈旧条目 | value 置 null + 重排探测链 |
| `InheritableThreadLocal` | 父子线程传递 | Thread.init 时复制快照 |

### 关键设计思想（编号列表）

1. **数据放线程，不放对象**：ThreadLocal 只是 key——线程隔离靠"存储位置"天然实现。
2. **弱引用防 key 泄漏**：key 不用强引用，外部一断引用 GC 就能收——结构上先防一半。
3. **懒清理换效率**：不设后台线程扫 Map，只在 set/get/remove 路径上顺带清——零闲置成本。
4. **结构性残留要靠契约**：value 强引用是泄漏根源，所以"用完 remove"是硬契约不是建议。
5. **池化放大问题**：线程复用让残留值"被下一个任务读到"——串号与泄漏是同源的。

### 面试题 Q&A

**Q1：ThreadLocal 的内存泄漏是怎么发生的？为什么弱引用没防住？**
> 结构：Thread → threadLocals（强）→ Entry[]（强）→ Entry（强）→ value（强）；Entry 的 key 是 WeakReference。泄漏链：外部把 ThreadLocal 置 null 后，key（弱引用）被 GC 回收，但 value 仍被"Thread → Map → Entry"强引用链持有。只要线程活着（线程池线程常驻）且没有 set/get/remove 触发懒清理，value 就永久残留。实测：1MB payload 置 null + GC 后，Map 里 key==null 的 Entry=1——key 没了，value 还在。弱引用只防住了 key 的泄漏，value 的清理必须靠 remove。

**Q2：为什么线程池里 ThreadLocal 会"串号"？怎么根治？**
> 线程池线程复用且常驻，Thread.threadLocals 不销毁；task1 set 的值残留在线程的 Map 里，task2 复用同一线程 get 到残留值。实测：task1 set "user-1001"，task2 get 到 "user-1001"。根治：set 之后在 finally 里 remove()，或用 AOP/任务包装器统一在任务结束后清理。口诀"set 一次必配 remove 一次"。

**Q3：ThreadLocalMap 的清理机制是怎样的？为什么不自动清 value？**
> 懒清理：只在 set/get/remove 路径上顺带清理陈旧 Entry（key==null）。getEntryAfterMiss 探测时遇到陈旧条目调 expungeStaleEntry（value 置 null + 重排探测链）；set 前 cleanSomeSlots；扩容（size >= len*2/3）前 expungeStaleEntries 全清。不自动清的原因：GC 管不到业务语义，"这个 key 再也不用了"只有代码知道；且后台线程扫 Map 成本高、与"线程局部"的定位冲突。所以 set 后不再访问的短任务会永久残留，必须 remove。

**Q4：InheritableThreadLocal 有什么坑？**
> ① 只在子线程创建瞬间复制一次快照——父线程之后改值子线程看不到；② 线程池场景：池线程是"创建它那次"的父线程快照，复用后值会串。生产传递上下文优先显式传参或任务包装器。

---

> **互链**：本篇为 [09-并发编程.md](09-并发编程.md) 的 `7.1` 块机器层深挖；线程池的线程复用机制（串号的根源）见 `09-源码解析-线程池ThreadPoolExecutor.md` 5.2；弱引用相关（Reference 体系与 GC）见 [15-JVM 原理]（后续主题）；ThreadLocal 在并发容器/异步编排里的配套清理可参考 `09-并发编程.md` 7.3/6.4 块。
