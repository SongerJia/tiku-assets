# 09 源码解析 · ThreadLocal 内存泄漏（机器层深挖）

> 配套主题文档《09 并发编程》块〔31〕。主题讲"原理与串号踩坑"，本篇扒 OpenJDK 8 源码看"弱引用到底弱在哪、泄漏怎么发生、为什么线程池最危险"。

## `ThreadLocal` 存在哪：每个线程一张 `ThreadLocalMap`

很多人误以为 `ThreadLocal` 自己存了值——**错**。值存在**线程对象**里：

```java
// Thread.java
ThreadLocal.ThreadLocalMap threadLocals = null; // 每个 Thread 一个
```

`ThreadLocal` 只是个"钥匙"。`set` 时：`Thread t = Thread.currentThread(); t.threadLocals.set(this, value)`。`get` 时拿当前线程的 map，用 `ThreadLocal` 实例当 key 取 value。所以**线程销毁，map 才销毁，值才释放**——这是后面所有坑的根源。

## Entry 为什么是弱引用：key 会丢但 value 不丢

`ThreadLocalMap` 不是 `HashMap`，而是**自定义哈希表（开放寻址）**，节点 `Entry`：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;
    Entry(ThreadLocal<?> k, Object v) {
        super(k);      // key 是弱引用！
        value = v;     // value 是强引用！
    }
}
```

**关键设计**：key（`ThreadLocal` 本身）是 `WeakReference`，value 是普通强引用。为什么要弱引用 key？为了让"业务里不再用的 `ThreadLocal`"能被 GC 回收，不至于因为 map 还引用着而泄漏 **key**。但**副作用**：GC 后 key 变 `null`，而 value 仍是强引用、且没别的引用 → **value 成了"有值无键"的孤儿**，这就是泄漏点。

## `set` 的开放寻址与过期清理（expungeStaleEntry）

`ThreadLocalMap` 用**线性探测**解决哈希冲突：`hash = threadLocalHashCode & (len-1)`，冲突就 `i++` 找下一个空位。

```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length, i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) { e.value = value; return; }  // 同 key 覆盖
        if (e != null && e.get() == null) {               // 发现过期 entry（key 被 GC）
            replaceStaleEntry(key, value, i);              // 顺手清理+复用
            return;
        }
    }
    tab[i] = new Entry(key, value);
    if (++size > threshold) rehash();
}
```

`replaceStaleEntry` → `expungeStaleEntry` 会把"key==null 的过期 entry"的 `value` 也置 `null`、slot 清空。**重要结论**：`set`/`get` 会**顺手清理**过期 entry，所以"只要还在调 `set/get`，泄漏会被逐步打扫"。但——如果设完值**再也不碰这个 ThreadLocal**，清理永远不会触发。

## 内存泄漏到底怎么发生的

完整泄漏链条：
1. 线程设 `tl.set(bigObj)` → map 里 Entry(key=tl弱引用, value=bigObj强引用)。
2. 业务不再用 `tl`，且 `tl` 这个变量也没别处引用 → GC 时 **key 被回收**（弱引用），Entry 变 `(null, bigObj)`。
3. **value=bigObj 仍是强引用**，且 Entry 还在 map 数组里。只要线程活着，bigObj 就 freeing 不了。
4. 若线程**很快结束**（如 `new Thread().start()` 跑完），map 随线程死亡，value 一起释放——**不泄漏**。
5. 若线程**常驻**（线程池！），map 永远在，bigObj 永远占着内存 → **真泄漏**。

所以泄漏的两个前提：**weak key 被 GC + 线程不销毁 + 之后不再 set/get 触发清理**。

## 线程池场景下为什么会"串号"（越权事故）

比泄漏更致命的是**串号**。线程池线程复用：请求 A 在同一线程上 `tl.set("userA")` 处理后**没 `remove()`**，线程被池回收；请求 B 拿到同一线程，`tl.get()` 返回 `"userA"`——B 以为自己是 A，**越权访问 A 的数据**。生产事故真实发生过（上下文透传漏 remove → 用户看到别人订单）。

```java
// 危险：用完不 remove
private static final ThreadLocal<String> UID = new ThreadLocal<>();
void handle(Request req) {
    UID.set(req.userId);
    biz(req);   // 若这里抛异常或忘了 remove → 下一请求串号
}
// 正确
void handle(Request req) {
    try { UID.set(req.userId); biz(req); }
    finally { UID.remove(); }   // 必须！同时防泄漏 + 防串号
}
```

## `remove` 为什么必须写进 finally

`remove()` 直接删 entry 并 `expungeStaleEntry` 清理邻居：

```java
public void remove() {
    ThreadLocalMap m = Thread.currentThread().threadLocals;
    if (m != null) m.remove(this); // 同时把 value 置 null
}
```

它一步到位：既切断 value 强引用（防泄漏），又保证下次 `get` 拿不到旧值（防串号）。**放 `finally`** 是因为不管业务正常返回还是抛异常，`finally` 都执行——这是线程池场景的强制纪律。主题文档块〔31〕的口诀"用完不 remove 等于埋雷"就是从这条源码推出来的。

## `InheritableThreadLocal` 与线程池的坑

`InheritableThreadLocal` 在**创建子线程时**把父线程的 value 拷贝给子线程（构造 `Thread` 时 `init` 会 `inheritThreadLocals`）。看着美好，但**线程池里完全失效**：池里的线程是早就建好的，不是"请求时 new 出来的子线程"，父请求的 value 根本不会传给池线程。要在线程池里透传上下文，得用阿里开源的 **`TransmittableThreadLocal`（TTL）**——它在线程"借出/归还"时做 value 的快照拷贝。这是框架级坑，很多链路追踪（traceId）透传 bug 源于此。

## 一句话收尾

ThreadLocal 的泄漏不在"弱引用"本身，而在"弱 key + 强 value + 线程常驻 + 不再 set/get"四件套凑齐。弱引用只是让 key 先死，value 的命得靠 `remove()` 来保。线程池场景里，漏 `remove()` 同时引爆"泄漏"和"串号"两颗雷。小张一句到位——`ThreadLocal` 是上下文利器，也是线程池里最容易被忽略的炸弹，`try/finally remove` 是唯一的拆弹钳。
