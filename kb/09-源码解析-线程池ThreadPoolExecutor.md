# 09 源码解析 · 线程池 ThreadPoolExecutor（机器层深挖）

> 配套主题文档《09 并发编程》块〔27〕〔28〕〔30〕。主题讲"七参数与流程"，本篇扒 OpenJDK 8 源码看"任务怎么被接、线程怎么被造、空闲怎么被回收"。

## `ctl` 一个 int 怎么同时装下"运行状态+线程数"

`ThreadPoolExecutor` 用**一个 `AtomicInteger ctl`** 同时编码两类信息（经典位运算技巧）：

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;        // 29 位
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;  // 约 5 亿
// 高 3 位 = 运行状态，低 29 位 = 工作线程数
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
```

5 种状态（从大到小）：`RUNNING(-1) > SHUTDOWN(0) > STOP(1) > TIDYING(2) > TERMINATED(3)`。用 `compareAndSet` 原子改 ctl，避免"状态"和"线程数"两字段需要额外同步。**设计精髓**：把两个经常一起变、又要求原子可见的变量压进一个 int，CAS 一把搞定。

## `execute` 的三步决策链（源码逐行）

```java
public void execute(Runnable command) {
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {       // ① 线程数 < 核心 → 直接建核心线程
        if (addWorker(command, true)) return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) { // ② 否则入队列
        int recheck = ctl.get();
        if (!isRunning(recheck) && remove(command))  // 入队后复查：若已 SHUTDOWN，回滚
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);                  // 没线程了，拉一个起来消费队列
    }
    else if (!addWorker(command, false))            // ③ 队列满 → 建非核心线程（到最大）
        reject(command);                             // ④ 连最大也超 → 拒绝
}
```

这正好对应主题文档〔28〕的流程图：**核心 → 队列 → 最大 → 拒绝**。注意 ② 的"入队后复查"是防御 `SHUTDOWN` 期间 Race：刚判断还能跑，入队瞬间池被关了，就把任务踢出并拒绝。

## `Worker` 是什么：既是线程又是个 AQS 锁

`Worker` 是内部类，**既实现 `Runnable`（线程跑的就是它），又继承 `AbstractQueuedSynchronizer`**：

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    final Thread thread;
    Runnable firstTask;
    Worker(Runnable firstTask) {
        setState(-1); // 初始锁住，runWorker 启动后才解锁（防止 start 前被中断）
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }
    protected boolean isHeldExclusively() { return getState() != 0; }
    public void run() { runWorker(this); }
}
```

Worker 用 AQS 的 `state`（0/1）当"是否正在运行任务"的锁：`lock()` 在真正执行 task 前加，`unlock()` 后释放，这样 `shutdown()` 时能用 `tryLock` 判断"这个 Worker 正忙着还是闲着"。**讽刺又巧妙**：线程池自己底层也靠 AQS。

## `runWorker`：线程启动后到底在干嘛

`addWorker` 启动 `worker.thread` → 跑 `Worker.run` → `runWorker`：

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // 解初始锁（state -1 → 0）
    try {
        while (task != null || (task = getTask()) != null) { // 不断取任务
            w.lock();
            try {
                beforeExecute(wt, task);
                task.run();             // 真正执行（在当前 Worker 线程）
                afterExecute(task, null);
            } finally {
                task = null;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly); // 线程退出，从 workers 摘掉
    }
}
```

核心循环：**有任务就跑，没任务就 `getTask()` 去队列/超时等**。所以"线程复用"= 一个 `while` 循环反复 `task.run()`，`run()` 结束不销毁线程，而是接着取下个任务。

## `getTask`：空闲线程如何被回收（keepAlive 真相）

`getTask()` 决定 Worker 是"阻塞等任务"还是"超时退出被回收"：

```java
private Runnable getTask() {
    boolean timedOut = false;
    for (;;) {
        int c = ctl.get();
        if (runStateAtLeast(c, SHUTDOWN) && (queue.isEmpty())) return null; // 关池且空 → 退出
        int wc = workerCountOf(c);
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize; // 非核心(或可超时核心)才计时
        if ((wc > maximumPoolSize || (timed && timedOut)) && (wc > 1 || queue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c)) return null;        // 回收一个线程
            continue;
        }
        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) : // 超时等
                workQueue.take();                                       // 永久等（核心线程）
            if (r != null) return r;
            timedOut = true;
        } catch (InterruptedException retry) { ... }
    }
}
```

**真相**：核心线程用 `workQueue.take()` **永久阻塞**不回收；非核心线程用 `poll(keepAliveTime)` **超时回收**。所以 `keepAliveTime` 只对"超出核心的线程"有效，除非 `allowCoreThreadTimeOut(true)`。`workQueue` 为空且池关闭时线程直接退出。

## 拒绝策略在源码里何时触发

拒绝只在 `execute` 第 ④ 步 `addWorker(command, false)` 返回 `false` 时调 `reject(command)`。而 `addWorker` 返回 false 的条件是：`wc >= (核心或最大上限)` 或池已非 RUNNING。`RejectedExecutionHandler` 四个实现：`AbortPolicy` 抛异常、`CallerRunsPolicy` 调 `r.run()`（提交方自己跑）、`DiscardPolicy` 啥也不做、`DiscardOldestPolicy` 丢队列头再 `execute`。实测（核心2/最大4/队列2，提交10 → 接受6 拒绝4）正是第 ④ 步触发。

## `shutdown` vs `shutdownNow` 差在哪

- `shutdown()`：CAS 把状态置 `SHUTDOWN`（不接受新任务，但跑完队列里已有的），中断**空闲** Worker（`interruptIdleWorkers`），等所有任务结束转 `TERMINATED`。
- `shutdownNow()`：置 `STOP`，**中断所有** Worker（包括正在跑的），`drainQueue()` 把队列里未跑的任务全部返回给你。粗暴但快。

**踩坑**：`shutdown()` 后若任务里又 `submit` 新任务会抛 `RejectedExecutionException`；优雅停池要 `shutdown()` + `awaitTermination` 兜底 + 超时 `shutdownNow`。

## 为什么默认线程池会 OOM（队列无界溯源）

`Executors.newFixedThreadPool` 的 `workQueue = new LinkedBlockingQueue<Runnable>()`，**构造没传 capacity**，默认 `Integer.MAX_VALUE`（约 21 亿）。回看 `execute` 第 ② 步：只要 `workerCountOf(c) >= corePoolSize` 就直接 `workQueue.offer`，队列"几乎永远不满"，于是第 ③ 步扩线程、第 ④ 步拒绝**永远走不到**。任务只进不出地堆队列 → 内存爆。`newCachedThreadPool` 则相反：队列是 `SynchronousQueue`（容量0），`offer` 必失败，于是疯狂 `addWorker` 到 `Integer.MAX_VALUE` 线程。两者都是一个"参数失控"极端。所以〔29〕强调：**永远手写 `ThreadPoolExecutor` + 有界队列 + 显式拒绝策略**。

## 一句话收尾

线程池的本质 = `ctl`(状态+计数) + `Worker`(线程壳+AQS锁) + `getTask`(取任务/超时回收) 的协作。`execute` 的三步决策和 `getTask` 的 `take/poll` 分叉，精确解释了"核心线程常驻、非核心超时回收、队列满才拒绝"。小张一句到位——线程池不是黑盒，它就是把"建线程、排队、回收"用一段 `while` 循环管起来的管家。
