# 09 源码解析 · AQS 与 ReentrantLock（机器层深挖）

> 配套主题文档《09 并发编程》块〔23〕〔24〕。主题文档讲"AQS 是什么、怎么用"，本篇扒 OpenJDK 8 源码看"它怎么实现"。结论先行：AQS 用**一个 volatile int 状态 + 一条 CLH 双向等待队列 + 模板方法**把"排队、阻塞、唤醒"全部托管，子类只填 `tryAcquire/tryRelease` 几个钩子。

## AQS 到底管了什么：state、队列、Node 三件套

`AbstractQueuedSynchronizer` 核心字段就三个：

```java
private volatile int state;          // 同步状态，语义由子类定
private transient volatile Node head;// 等待队列头（哑节点）
private transient volatile Node tail;// 等待队列尾
```

`Node` 是队列节点，关键字段：`volatile int waitStatus`（CANCELLED=1 / SIGNAL=-1 / CONDITION=-2 / PROPAGATE=-3 / 0）、`volatile Node prev/next`、`volatile Thread thread`、`Node nextWaiter`（CONDITION 队列用）。

**设计要点**：`state` 用 `volatile` + CAS 改（保证原子与可见）；队列是**变体 CLH**（Craig-Landin-Hagersten）FIFO 锁，入队用 CAS 改 `tail`，出队改 `head`。AQS 自己不实现"锁语义"，只实现"谁抢不到就排队 park"的通用骨架。

## `acquire` 流程：抢锁失败如何入队挂起

`acquire(int arg)` 是独占获取入口，源码极短却暗藏玄机：

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&            // 1. 先抢（子类实现）
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) // 2. 抢失败入队自旋
        selfInterrupt();
}
```

`addWaiter` 把当前线程包成 Node，CAS 接到 `tail`；`acquireQueued` 在队列里**自旋**：若自己是 `head.next`（队首），再试一次 `tryAcquire`，成功就当 head 出队；失败就 `shouldParkAfterFailedAcquire` 把前驱 `waitStatus` 设成 `SIGNAL`，然后 `parkAndCheckInterrupt()` 调 `LockSupport.park()` **挂起**。被 `unpark` 唤醒后继续自旋抢。这种"自旋+CAS 入队+park"避免了无谓的上下文切换。

## `release` 流程：释放后如何唤醒后继

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {              // 子类释放状态
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);         // 唤醒 head 的后继
    }
    return true;
}
```

`unparkSuccessor` 从 `head.next` 找第一个 `waitStatus <= 0` 的节点，`LockSupport.unpark(node.thread)` 唤醒它。被唤醒的线程在 `acquireQueued` 里重新抢锁。注意：**只唤醒一个**（独占模式），这是为什么 ReentrantLock 公平/非公平都保证"一个线程拿到锁"。

## `tryAcquire` 在 ReentrantLock 里怎么写（非公平 vs 公平）

`ReentrantLock` 的 `NonfairSync.tryAcquire` 直接调 `Sync.nonfairTryAcquire`：

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) { // 不管排队，直接抢——非公平插队
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;               // 重入：state++
        setState(nextc);
        return true;
    }
    return false;
}
```

**公平锁 `FairSync.tryAcquire`** 多了一步：`if (hasQueuedPredecessors()) return false;`——队列里有人排前面就不抢，保证先来先得。**非公平**少了这步，新来的能插队，吞吐更高但可能饿死老线程。这就是〔20〕公平/非公平差异的源码根因。

## 可重入是怎么用 state 实现的

可重入 = 同一线程多次 `lock` 不阻塞。看上面代码：`else if (current == getExclusiveOwnerThread())` 命中时只 `state++`，不抢锁。释放时 `tryRelease`：

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) { free = true; setExclusiveOwnerThread(null); } // 到 0 才真释放
    setState(c);
    return free;
}
```

**必须 `lock`/`unlock` 配对**——每多一次 `lock` 就 `state++`，只有 `state` 减回 0 才清 owner、允许别的线程抢。少 `unlock` 一次 = 锁永不释放 = 死锁。

## CLH 队列为什么是 FIFO 且 head 是哑节点

队列初始化时 `head == tail == null`；第一个线程抢锁成功**不入队**（直接持锁，head 仍是 null）。等它持锁时第二个线程来抢失败，`addWaiter` 创建 `head` 作为**哑节点（dummy）**，再把真正等待的节点挂到 `head` 后面。所以 `head` 永远是"已拿到锁或已出队"的占位，**真正等待的是 `head.next` 开始**。

FIFO 保证：入队一律 CAS 接 `tail`，唤醒一律从 `head.next` 起。公平锁靠 `hasQueuedPredecessors` 不插队，非公平靠允许插队——但**一旦入队就严格 FIFO**，避免队列内饥饿。

## `LockSupport.park/unpark` 底层是什么

`park/unpark` 是 `Unsafe` 对 OS 原语的封装：Linux 上基于 `pthread_mutex` + `pthread_cond`（或 `futex`）。`park()` 让线程**阻塞在用户态可中断点**，比 `synchronized` 的重量级 Monitor 更轻（不进内核排队，直接 futex 等待）。`unpark(thread)` 提前给"许可"，若线程还没 park，下次 park 立刻返回——这点很关键，`release` 唤醒时线程可能还没 park，unpark 不会丢失。

**对比 synchronized**：内置锁等不到进 `_EntryList` 由 OS Monitor 管，涉及更重的上下文切换；AQS/LockSupport 用 futex，竞争低时几乎无切换成本。

## 独占 vs 共享模式差在哪

`acquire`/`release` 是独占（锁）；`acquireShared`/`releaseShared` 是共享（如 Semaphore 许可、CountDownLatch 计数）。**最大差异在释放唤醒**：独占 `unparkSuccessor` 只唤醒一个后继；共享 `doReleaseShared` 用 `PROPAGATE` 状态**链式唤醒**——一个节点被唤醒并获取到共享资源后，若还有剩余，会继续 `unpark` 下一个，直到资源耗尽或队列走完。这就是 CountDownLatch `countDown` 到 0 能一次性放所有 `await` 线程的原因（见〔24〕）。

## 一句话收尾

AQS 把"并发同步"抽象成**状态竞争 + 排队阻塞 + 唤醒**三件事，用模板方法让 ReentrantLock/Semaphore/CountDownLatch 只填业务逻辑。看懂 `state` 的语义（次数/许可/计数）和 CLH 队列的 `SIGNAL` 唤醒协议，JUC 一半的类就通了。小张一句到位——AQS 不是锁，是"造锁的工厂模具"。
