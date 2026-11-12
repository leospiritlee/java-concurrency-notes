# 第8章 并发容器选型：别再无脑 `synchronizedList`

**一句话**：JUC 提供了一整套为并发设计的容器，选型看"读多写少 / 写多 / 需要阻塞 / 需要顺序"——用错容器比加锁更低效甚至更错。

**为什么需要它**：现场：你给 `ArrayList` 套个 `synchronizedList` 做缓存，读多写少下每次读都抢锁，吞吐惨淡。现场二：你用普通 `HashMap` 在多线程下 `put`，偶发数据丢失甚至死循环（JDK7 头插法）。现场三：生产者消费者你手写了 `wait/notify` 队列，bug 不断——其实该用阻塞队列。现场四：你以为 `ConcurrentHashMap` 的 `get` 结果一定最新——它在弱一致性下可能读到稍旧的值。这一章给出一张"按场景选容器"的判断表。

**核心结构**：

| 容器 | 适用 | 注意 |
|---|---|---|
| `ConcurrentHashMap` | 高并发 KV，读多写多 | 弱一致性迭代、size 是估计 |
| `CopyOnWriteArrayList` | 读远多于写（监听器、配置） | 写贵、读有短暂延迟 |
| `ConcurrentLinkedQueue` | 高并发非阻塞队列 | 无阻塞、size 慢 |
| 阻塞队列家族 | 生产者-消费者 | 满/空时阻塞或超时 |
| `Collections.synchronized*` | 简单兼容、低并发 | 全方法加锁，吞吐低 |

**关键设计**：

```text
  怎样选并发容器（决策）

  需要 KV 映射？
    └→ ConcurrentHashMap（默认首选）
  需要线性表？
    读 >> 写？ → CopyOnWriteArrayList
    读写均衡且要线程安全？ → synchronizedList（低并发）或分片
  需要队列？
    生产者-消费者？ → 阻塞队列（Array/Linked/Synchronous/Priority/Delay）
    高并发无阻塞？ → ConcurrentLinkedQueue
```

**拆解**：

### 8.1 ConcurrentHashMap：高并发 KV 的默认选择
`ConcurrentHashMap` 用分段/树化 + CAS + 细粒度锁，把并发冲突降到最低，读基本无锁。但它提供的是**弱一致性**：迭代器反映某一时刻的快照、不抛 `ConcurrentModificationException`、`size()` 是估计值。这意味着"遍历过程中其他线程的修改，你可能看到也可能看不到"——对"计数必须精确"的场景（如实时在线人数），要靠 `LongAdder` 或原子计数，别依赖它的 `size()`。

### 8.2 CopyOnWrite：读多写少的特例
`CopyOnWriteArrayList`/`CopyOnWriteArraySet` 每次修改复制底层数组，读完全无锁且看到稳定快照（见第 5 章）。写极贵、读可能短暂滞后。只对"读远多于写、能容忍短时不一致"有效——监听器列表、配置项、白名单都是典型。把它用在写频繁的缓存上就是灾难。

### 8.3 阻塞队列家族：生产者-消费者的标准件
`BlockingQueue` 把"队列空时消费者等、满时生产者等"封装好了，免去了手写 `wait/notify` 的痛苦：
- `ArrayBlockingQueue`：有界、固定数组，最常用。
- `LinkedBlockingQueue`：可选有界，链表实现，吞吐通常更好。
- `SynchronousQueue`：不存储元素，递交即 handoff，配合 `newCachedThreadPool`。
- `PriorityBlockingQueue`：带优先级。
- `DelayQueue`：按延迟时间出队，做定时/重试很方便。
生产者-消费者模型（第 11 章）几乎总该基于阻塞队列，而不是自己造轮子。

### 8.4 非阻塞与 synchronized 包装
`ConcurrentLinkedQueue` 用 CAS 实现无锁并发队列，高并发下竞争低但 `size()` 要遍历、较慢。低并发兼容场景用 `Collections.synchronizedXxx` 即可（全方法加锁，简单但吞吐低）。关键判断：**先想清"读写比例和是否需要阻塞"，再选容器**——绝大多数 KV 用 CHM、绝大多数生产者消费者用阻塞队列、绝不要拿普通 `HashMap`/`ArrayList` 直接多线程用。

**今天怎么做**：容器选型原则与 JDK 版本无关（CHM 在 JDK8 后实现大改但 API 不变）。响应式/ actor 模型下可能改用不可变集合或专门数据结构，但"按读写特征选型"的判断不变。注意：这些容器解决"数据结构本身的并发安全"，不解决"多个操作之间的业务原子性"——"`if (!map.containsKey(k)) map.put(k,v)`"在 CHM 下仍非原子，要用 `putIfAbsent`/`compute`。

<div class="chapend" markdown="1">

**容易误解的地方**：

1. 以为"`ConcurrentHashMap` 的 `size()` 精确"——它是估计值，精确计数请用专用原子计数。
2. 以为"CopyOnWrite 读写都快"——写极慢，只适合读碾压写的场景。
3. 以为"普通 List 套 synchronized 就行"——低并发 OK，高并发吞吐差，且 `HashMap` 并发 `put` 有数据损坏风险。
4. 以为"用了并发容器业务就原子了"——多步操作（检查再放）仍要 `putIfAbsent`/`compute` 这类原子 API。

**记住这一句**：并发容器按"读写比例 + 是否需要阻塞"选型；CHM 管 KV、阻塞队列管生产者消费者，多步操作仍要原子 API。

</div>
