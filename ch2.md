# 第2章 可见性：volatile 与内存屏障

**一句话**：`volatile` 保证"写立即可见、读必取最新"，并禁止特定重排——但它不保证原子性，复合操作还得靠锁或原子类。

**为什么需要它**：现场：你用一个 `boolean running = true` 控制线程退出，主线程设 `running = false`，工作线程却一直不退出——因为 `running` 没声明 `volatile`，工作线程一直在读自己的缓存副本。现场二：你给计数器加了 `volatile`，结果并发自增还是少算——因为 `volatile` 管不了"读-改-写"的原子性。这两个现场分别对应 volatile 的"能"和"不能"。这一章讲清 volatile 到底保证什么、不保证什么，以及它背后的 happens-before 与内存屏障。

**核心结构**：

| 机制 | 提供 | 不提供 |
|---|---|---|
| `volatile` 写/读 | 可见性 + 禁止重排（特定） | 原子性（复合操作不安全） |
| `synchronized` | 原子性 + 可见性 + 有序性 | 可能阻塞、影响吞吐 |
| `final` 字段 | 安全发布（构造后可见且有序） | 运行期可变性 |
| happens-before | 一组"前者对后者可见且有序"的规则 | 不是锁，是约束关系 |

**关键设计**：

```text
  volatile 的两种典型正确用法

  ① 状态标志（单写多读，只写 Boolean/int，不复合）
     volatile boolean running = true;
     // 写线程：running = false;  读线程立刻看到 → 退出

  ② 双重检查锁里的引用（禁止"引用赋值"重排到"构造"之前）
     class Singleton {
       private static volatile Singleton inst;
       static Singleton get(){
         if (inst == null) {                     // 第一次检查
           synchronized (Singleton.class){
             if (inst == null) inst = new Singleton(); // volatile 防重排
           }
         }
         return inst;
       }
     }

  反例（volatile 救不了）：
     volatile int count;
     count++;          // 读-改-写 三段，volatile 不保证这三段原子
```

**拆解**：

### 2.1 volatile 保证的两件事
一是**可见性**：对 volatile 变量的写，会立刻刷新到主存；后续任意线程的读，都会从主存重新加载，不会一直用旧副本。二是**有序性（受限）**：volatile 禁止特定重排——具体说，它插入了内存屏障（memory barrier），使得" volatile 写之前的操作"不会被重排到写之后，" volatile 读之后的操作"不会被重排到读之前。这正是双重检查锁需要它的原因：防止"先把对象引用暴露出去、对象却还没构造完"。

### 2.2 happens-before：把"可见+有序"形式化
JMM 用 happens-before 规则描述"什么情况下一个操作的结果对另一个操作可见且有序"。关键规则：
- **程序次序**：同一线程内，前面的操作 happens-before 后面的操作。
- **volatile 规则**：对 volatile 变量的写 happens-before 后续对该变量的读。
- **锁规则**：`unlock` happens-before 后续对同一锁的 `lock`。
- **传递性**：A hb B、B hb C，则 A hb C。
记住：happens-before 不是"时间上更早"，而是"JMM 保证可见且有序"的偏序关系。有了它，你判断"线程 B 能不能看到线程 A 的写"，就变成判断两者之间有没有一条 happens-before 链。

### 2.3 volatile 不保证原子性：复合操作的陷阱
这是最高频的误解。`volatile int count; count++;` 依然不安全——`++` 是读-改-写三步，volatile 只保证"每一步看到最新值"，但不保证"三步之间不被打断"。多个线程可能在各自读到同一个旧值后各自 +1，互相覆盖。要用原子类（`AtomicInteger`）或锁来保护复合操作。一句话判断：**volatile 适用于"单个变量、单次读写"的可见性；涉及"多个变量"或"读-改-写"就必须另寻原子性手段。**

### 2.4 64 位 long/double 与 final 的发布语义
Java 规定 32 位读写为原子，但 `long`/`double` 的普通读写在旧内存模型下可能被拆成两个 32 位操作（撕裂读）。`volatile long/double` 强制原子读写。另一个常被忽视的是 `final`：正确发布的 `final` 字段，JMM 保证"构造完成后对其他线程可见且构造中的写入不会重排到构造外"——这正是"用 final 修饰不可变对象的字段"能实现安全发布的底层原因（见第 5 章）。

**今天怎么做**：volatile 的语义在 Java 5 之后的 JMM 里已经稳定且严格，无需"为了保险再套一层锁"。现代写法里，状态标志、懒加载引用、独立观测值用 volatile；复合状态用 `java.util.concurrent.atomic` 包。注意：volatile 不能替代锁来保护一段临界区，它只保护"单个变量"。

<div class="chapend" markdown="1">

**容易误解的地方**：

1. 以为"`volatile` 是轻量锁"——它不是锁，不保证原子性，也不互斥，只是可见性+有序性约束。
2. 以为"volatile 能修好 `i++`"——复合操作缺的是原子性，volatile 给不了。
3. 以为"happens-before 就是时间先后"——它是可见性/有序性的保证关系，不是时钟顺序。
4. 以为"加了 volatile 就线程安全了"——只有当共享状态是单一变量的单次访问时才够，多变量或复合操作不行。

**记住这一句**：volatile 负责"看得见、不乱序"，不负责"一气呵成"；单变量单次访问用它，读-改-写和多个变量交给锁或原子类。

</div>
