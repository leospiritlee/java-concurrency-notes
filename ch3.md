# 第3章 原子性：synchronized 与显式锁的取舍

**一句话**：`synchronized` 够用且省心，但拿不到超时/中断/多条件/`tryLock`；需要这些时再用 `ReentrantLock`，并务必在 `finally` 里 `unlock`。

**为什么需要它**：现场：你的方法加了 `synchronized`，结果高并发下整体变慢——因为锁粒度太粗，把不相干的操作也圈进去了。现场二：你想"尝试获取锁、拿不到就做别的事"，`synchronized` 做不到——它没有 `tryLock`、不支持超时和中断。现场三：读多写少，用了独占锁，读之间也互相阻塞，吞吐上不去——需要读写锁。现场四：忘了在 `finally` 释放显式锁，异常路径下锁永远不释放，线程全卡死。这一章把"用哪个锁、为什么、代价是什么"讲清。

**核心结构**：

| 手段 | 能力 | 代价/注意 |
|---|---|---|
| `synchronized` | 原子性+可见性+有序性，JVM 自动加解锁 | 不可超时、不可中断、不能多条件、锁对象唯一 |
| `ReentrantLock` | 上述 + `tryLock`/超时/中断/公平/多 `Condition` | 必须 `finally` 里 `unlock` |
| `ReadWriteLock` | 读共享、写独占 | 写饥饿风险；`StampedLock` 提供乐观读 |
| `StampedLock` | 读多写少场景的乐观读，几乎无读开销 | API 偏底层，用错易死锁 |

**关键设计**：

```java
// 显式锁的最小正确范式：try-finally 释放
private final ReentrantLock lock = new ReentrantLock();
void update() {
  lock.lock();                 // 或 lock.tryLock(100, MS)
  try {
    // 临界区
  } finally {
    lock.unlock();             // 异常也必须释放，否则死锁
  }
}

// 读写锁：读之间不互斥
private final ReadWriteLock rw = new ReentrantReadWriteLock();
void read()  { rw.readLock().lock();  try{...} finally{ rw.readLock().unlock(); } }
void write() { rw.writeLock().lock(); try{...} finally{ rw.writeLock().unlock(); } }
```

注意 `synchronized` 不需要手动释放——JVM 在方法/代码块退出（含异常）时自动释放，这是它"省心"的根本。

**拆解**：

### 3.1 synchronized 的适用面与边界
`synchronized` 提供原子性、可见性、有序性三位一体，且由 JVM 托管加解锁，几乎不会因"忘记释放"而出事。它的局限是能力固定：获取锁时不能设超时、不能被中断、拿不到就只能一直等；一个锁只有一把"条件队列"，复杂等待/通知场景不够用；且锁对象必须是同一个实例，配错对象（如锁了不同 String 常量或 new 出来的对象）会完全失效。在这些局限不影响你时，优先用 `synchronized`——简单即正确。

### 3.2 何时升级到 ReentrantLock
需要 `synchronized` 给不了的能力时，才上 `ReentrantLock`：
- **`tryLock(timeout)`**：拿不到锁就在超时后放弃或转降级，避免无限等待——这是实现"限流/快速失败"的关键。
- **可中断**：`lockInterruptibly()` 让等待锁的线程能响应中断。
- **多条件变量**：一个锁可以挂多个 `Condition`，分别 `await`/`signal`，比 `synchronized` 的单等待队列精细。
- **公平/非公平**：可显式选公平锁（按请求顺序，避免饥饿，但吞吐低）。
代价是必须自己 `try-finally` 释放，且心智负担更重——能不用则不用。

### 3.3 读多写少：读写锁与 StampedLock
当共享状态"读远多于写"，用独占锁会让所有读互相阻塞，浪费并发。读写锁（`ReentrantReadWriteLock`）允许多个读并发、写独占。但要注意写饥饿：读一直占着，写可能长期拿不到。StampedLock 进一步提供**乐观读**——读时不加锁，只在提交时校验戳记是否被写改过，没改过就零开销读完，极大提升读多写少吞吐。代价是 API 偏底层、用错（比如忘了校验戳）极易出 bug，只在确实有读多写少瓶颈时才值得。

### 3.4 锁粒度与性能陷阱
"加锁就安全"是错觉，真正决定正确性的是**锁保护了哪些状态、锁对象是否唯一、粒度是否合适**。锁粒度太粗（把无关 IO、日志圈进临界区）会严重损失并发；太细（多个锁保护关联状态）又可能漏掉组合不变式。另一个经典坑：用不同实例当锁对象，导致看似加锁实则各锁各的——锁对象必须是所有线程共享且唯一的（通常是 `static final` 或同一个实例字段）。与第 9 章的原子类相比，锁适合"临界区较长、逻辑复杂"，原子类适合"单一变量的简单更新"。

**今天怎么做**：`synchronized` 在 JVM 持续改进（偏向锁已于新版本默认弱化），多数业务场景仍首选。需要超时/中断/多条件才用 `ReentrantLock`；读多写少才考虑读写锁/`StampedLock`。无论哪种，临界区里**不要做可能抛出受检异常的外部调用后不释放**，显式锁务必 `finally` 释放。

<div class="chapend" markdown="1">

**容易误解的地方**：

1. 以为"`synchronized` 慢，能不用就不用"——现代 JVM 下它常常更快更稳，且不会忘释放。
2. 以为"显式锁更高级所以更好"——它只是多了能力，代价是必须手动释放，出错面更大。
3. 以为"读写锁读写完全不互斥"——写是独占的，且可能有写饥饿。
4. 以为"锁了方法就万事大吉"——锁对象不唯一种等于没锁，临界区过大则白白损失性能。

**记住这一句**：默认用 `synchronized`，只在需要超时/中断/多条件/读写分离时才升级到显式锁，且显式锁永远 `finally` 释放。

</div>
