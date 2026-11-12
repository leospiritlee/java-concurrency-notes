# 第7章 线程池与任务抽象：别再 `new Thread`

**一句话**：用 `Executor` 把"任务"和"执行任务的线程"解耦；线程池靠核心/最大线程数、队列、拒绝策略四维调参，选型错了要么耗尽资源要么丢任务。

**为什么需要它**：现场：你在循环里 `new Thread(r).start()`，流量一高系统被线程撑爆、上下文切换把 CPU 吃光。现场二：你用固定大小线程池，队列用无界 `LinkedBlockingQueue`，结果任务积压把内存打满 OOM。现场三：拒绝策略用了默认 `AbortPolicy`，高峰期直接抛 `RejectedExecutionException` 把请求打挂。现场四：你以为 `submit` 的异常会自己冒泡，结果默默吞掉，排错排到怀疑人生。这一章讲清线程池的运作模型、四个核心参数怎么配、以及 `Future`/`CompletableFuture` 怎么把异步编排写干净。

**核心结构**：

| 概念 | 作用 |
|---|---|
| `Executor` / `ExecutorService` | 任务提交与执行解耦的抽象 |
| 核心/最大线程数 | 常驻线程量 vs 峰值线程量 |
| 工作队列 | 暂存来不及执行的任务 |
| 拒绝策略 | 队列满且达最大线程时的兜底 |
| `Future` / `CompletableFuture` | 异步结果句柄与编排 |

**关键设计**：

```text
  ThreadPoolExecutor 的运转

  提交任务 → ① 核心线程未满？新建核心线程执行
            ② 否则入工作队列
            ③ 队列满且未达最大线程？新建非核心线程
            ④ 都满了 → 拒绝策略

  拒绝策略：Abort(抛异常) / Discard(静默丢) / DiscardOldest(丢最老) / CallerRuns(调用方线程自己跑)
```

**拆解**：

### 7.1 为什么不用 new Thread
每个 `new Thread` 都要分配栈、注册到 JVM、退出时回收，且数量无上限——高并发下线程数爆炸，OS 调度和内存都扛不住。线程池的核心思想是**复用固定数量的线程去执行大量短期任务**：线程创建一次、反复使用，任务以 `Runnable`/`Callable` 形式提交到队列，由空闲线程认领。这把"要做什么"（任务）和"谁来做、做几个"（线程管理）彻底分开。

### 7.2 四个核心参数怎么配
- **核心线程数**：常驻线程，即使空闲也保留（除非 `allowCoreThreadTimeOut`）。
- **最大线程数**：队列满后最多能扩到多少线程。
- **工作队列**：常见的有界 `ArrayBlockingQueue`（防 OOM）、无界 `LinkedBlockingQueue`（易积压）、`SynchronousQueue`（不缓存，直接交给线程，适合短任务高吞吐）。**无界队列会让最大线程数失效**——任务永远进队列，永远不新建非核心线程。
- **拒绝策略**：队列满且达最大线程时触发。默认 `AbortPolicy` 抛异常；`CallerRunsPolicy` 让提交任务的线程自己执行，起到"反向压制提交速度"的背压效果，常是更稳的选择。配置的本质是回答："队列多大、能扩到多少线程、满了怎么办"——没有万能值，取决于任务是 CPU 密集还是 IO 密集、能容忍延迟还是能容忍丢弃。

### 7.3 Future 与 CompletableFuture：异步结果怎么拿
`submit(Callable)` 返回 `Future`，用 `get()` 阻塞取结果或异常。`Future` 的表达力有限（一个结果、被动等待）。`CompletableFuture` 用 `thenApply`/`thenCompose`/`thenCombine` 把多个异步阶段组合成流水线，支持异常恢复（`exceptionally`/`handle`），是把"异步编排"写干净的主流方式。坑：Lambda 里抛的异常不会冒泡到调用栈，要靠 `get()` 或 `handle` 捕获，否则"看起来没报错实则失败了"。

### 7.4 ForkJoinPool 与 work-stealing
`ForkJoinPool` 用于"可分解的递归任务"（如大数组并行处理）：每个工作线程有自己的双端队列，干完自己的活会去"偷"别的线程队列尾部的任务（work-stealing），减少空闲。适合 CPU 密集、可拆分的任务；不适合管理阻塞的 IO。它也是 `CompletableFuture` 默认使用的池（除非显式指定 executor）。理解它的存在，能解释"为什么我的并行流有时快、有时反而慢"——任务粒度和是否阻塞决定了收益。

**今天怎么做**：`Executors` 提供的 `newFixedThreadPool`/`newCachedThreadPool` 是便捷封装，但隐藏了队列/线程数细节（缓存池用 `SynchronousQueue` 且无上限风险），生产上推荐**显式构造 `ThreadPoolExecutor`** 把参数摆上台面。虚拟线程（Project Loom）出现后，"每任务一线程"重新变回可行，但线程池的"限并发、背压、队列"思想在管理阻塞资源时依然必要。

<div class="chapend" markdown="1">

**容易误解的地方**：

1. 以为"无界队列最安全"——它把压力转嫁成内存，最终 OOM，且让最大线程数形同虚设。
2. 以为"默认拒绝策略(Abort)最严格所以最好"——高峰期它直接抛异常打挂调用方，CallerRuns 往往更稳。
3. 以为"submit 的异常会自己抛出来"——它被包进 Future，不 `get()` 就静默吞掉。
4. 以为"线程池越大吞吐越高"——线程过多会增加上下文切换，超过 CPU 核数后反而下降。

**记住这一句**：线程池用"复用 + 队列 + 拒绝策略"把任务与线程解耦；无界队列和默认拒绝策略是两个最常见的大坑。

</div>
