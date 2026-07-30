# 实操 02.2：手动配置线程池 & 死锁分析与排查

> ⏱ **预计时长**：60 分钟
> 📌 **难度**：⭐⭐⭐
> 🔧 **AI 辅助**：用 AI 生成线程池参数异常场景，辅助排查死锁和性能瓶颈

---

## 前置要求

- ✅ 已完成章节：02.2 线程安全 + synchronized + Lock、02.3 线程池 + JUC 工具
- ✅ 已完成实操：02.1 售票案例（熟悉线程安全问题的复现与解决）
- ✅ 需要安装：JDK 17+、Java IDE

## 实践目标

完成本次实操后，你将能够：

1. 手动创建带完整参数配置的线程池
2. 通过调整核心参数观察对程序性能的影响
3. 复现并排查死锁问题
4. 使用 Java 工具（jstack/jvisualvm）和 AI 辅助诊断并发故障

---

## 步骤

### Step 1：手动配置线程池

**操作说明：**

创建一个自定义线程池工厂，并根据不同的业务场景配置参数。

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadPoolFactory {
    /**
     * 创建自定义线程池
     *
     * @param poolName     线程池名称（用于命名线程）
     * @param coreSize     核心线程数
     * @param maxSize      最大线程数
     * @param queueCapacity 队列容量
     * @return 配置好的线程池
     */
    
    (
            String poolName, int coreSize, int maxSize, int queueCapacity) {

        return new ThreadPoolExecutor(
                coreSize,
                maxSize,
                60L,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(queueCapacity),
                new ThreadFactory() {
                    private final AtomicInteger counter = new AtomicInteger(1);

                    @Override
                    public Thread newThread(Runnable r) {
                        Thread t = new Thread(r);
                        t.setName(poolName + "-" + counter.getAndIncrement());
                        t.setDaemon(false);
                        t.setPriority(Thread.NORM_PRIORITY);
                        return t;
                    }
                },
                new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }
}
```

**public ThreadPoolExecutor(**

 int corePoolSize, // 1.核心线程数

 int maximumPoolSize, // 2.最大线程数 

long keepAliveTime, // 3.非核心线程空闲时长 

TimeUnit unit, // 4.时间单位 

BlockingQueue<Runnable> workQueue, // 5.阻塞任务队列 ThreadFactory threadFactory, // 6.线程工厂 RejectedExecutionHandler handler // 7.拒绝策略

**)**

### Step 2：分析核心参数对性能的影响

**操作说明：**

编写测试代码，分别测试不同参数组合下的任务执行情况。

```java
public class ThreadPoolParamTest {
    public static void main(String[] args) {
        System.out.println("=== 实验1：核心线程数对性能的影响 ===\n");
        testCorePoolSize();

        System.out.println("\n=== 实验2：队列容量的影响 ===\n");
        testQueueSize();

        System.out.println("\n=== 实验3：拒绝策略的影响 ===\n");
        testRejectionPolicy();
    }

    // 实验1：固定总任务数，改变核心线程数
    static void testCorePoolSize() {
        int totalTasks = 50;
        int[] coreSizes = {1, 2, 4, 8, 16};

        for (int coreSize : coreSizes) {
            ThreadPoolExecutor pool = ThreadPoolFactory.createPool(
                    "test-" + coreSize, coreSize, coreSize * 2, 100);

            long start = System.nanoTime();

            for (int i = 0; i < totalTasks; i++) {
                int taskId = i;
                pool.execute(() -> {
                    try {
                        Thread.sleep(100);  // 模拟耗时任务
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                });
            }

            pool.shutdown();
            try {
                pool.awaitTermination(60, TimeUnit.SECONDS);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }

            long duration = System.nanoTime() - start;
            System.out.printf("coreSize=%2d | 耗时: %6d ms | 平均: %4d ms/任务%n",
                    coreSize, duration / 1_000_000,
                    (duration / 1_000_000) / totalTasks);
        }
    }

    // 实验2：固定线程数，改变队列容量
    static void testQueueSize() {
        int[] queueSizes = {0, 10, 50, Integer.MAX_VALUE};
        // ... 类似实验1的实现，观察：
        // - 队列为 0 时是否会创建更多临时线程？
        // - 队列为 Integer.MAX_VALUE 时是否会 OOM？
    }

    // 实验3：不同拒绝策略的表现
    static void testRejectionPolicy() {
        // 用很小的线程池 + 很小的队列 + 提交大量任务
        // 分别测试 AbortPolicy / CallerRunsPolicy / DiscardPolicy
    }
}
```

**预期观察结果：**

```
实验1：核心线程数对性能的影响
coreSize= 1 | 耗时: 5023 ms | 平均: 100 ms/任务
coreSize= 2 | 耗时: 2518 ms | 平均:  50 ms/任务
coreSize= 4 | 耗时: 1280 ms | 平均:  26 ms/任务
coreSize= 8 | 耗时:  650 ms | 平均:  13 ms/任务
coreSize=16 | 耗时:  660 ms | 平均:  14 ms/任务 （过多的线程可能因上下文切换反而变慢）
```

> 💡 **AI 辅助用法**：将你的运行结果发给 AI，让 AI 分析为什么线程数增加到一定程度后性能不再提升（上下文切换开销）。

### Step 3：复现死锁

**操作说明：**

写一个经典的死锁案例，观察程序"卡死"的现象。

```java
public class DeadlockDemo {
    private static final Object RESOURCE_A = new Object();
    private static final Object RESOURCE_B = new Object();

    public static void main(String[] args) {
        // 线程1：持有 A，等待 B
        Thread t1 = new Thread(() -> {
            synchronized (RESOURCE_A) {
                System.out.println("线程1：持有资源 A，等待资源 B...");
                try { Thread.sleep(100); } catch (Exception e) {}

                synchronized (RESOURCE_B) {
                    System.out.println("线程1：获取到了资源 B");
                }
            }
        }, "线程-1");

        // 线程2：持有 B，等待 A（与线程1获取锁的顺序相反！）
        Thread t2 = new Thread(() -> {
            synchronized (RESOURCE_B) {
                System.out.println("线程2：持有资源 B，等待资源 A...");
                try { Thread.sleep(100); } catch (Exception e) {}

                synchronized (RESOURCE_A) {
                    System.out.println("线程2：获取到了资源 A");
                }
            }
        }, "线程-2");

        t1.start();
        t2.start();

        // 程序永远不会结束 → 死锁！
    }
}
```

**运行结果：**

```
线程1：持有资源 A，等待资源 B...
线程2：持有资源 B，等待资源 A...
（程序卡住，永不结束）
```

### Step 4：使用 jstack 排查死锁

**操作说明：**

1. 运行 DeadlockDemo，程序会卡住
2. 打开命令行，执行 `jps` 找到 Java 进程 ID
3. 执行 `jstack -l <进程ID>` 获取线程堆栈
4. 在输出中查找 "Found one Java-level deadlock" 部分

**预期输出：**

```
Found one Java-level deadlock:
=============================
"线程-1":
  waiting to lock monitor 0x0000012345 (object 0x...RESOURCE_B)
  which is held by "线程-2"
"线程-2":
  waiting to lock monitor 0x0000012345 (object 0x...RESOURCE_A)
  which is held by "线程-1"

Java stack information for the threads listed above:
...
```

> 💡 **AI 辅助用法**：将 jstack 输出的完整堆栈发给 AI，让 AI 帮你在复杂场景下分析死锁路径。

### Step 5：解决死锁

**方法1：固定锁获取顺序（推荐）**

```java
// 两个线程都用相同的锁顺序：先 A 后 B
// 线程1 和 线程2 都先锁 A 再锁 B → 不会死锁
```

**方法2：使用 tryLock 超时机制**

```java
public class DeadlockFreeDemo {
    private final Lock lockA = new ReentrantLock();
    private final Lock lockB = new ReentrantLock();

    public void safeMethod() throws InterruptedException {
        while (true) {
            // 尝试获取锁 A
            if (lockA.tryLock(1, TimeUnit.SECONDS)) {
                try {
                    // 尝试获取锁 B
                    if (lockB.tryLock(1, TimeUnit.SECONDS)) {
                        try {
                            // 两个锁都获取到了，执行业务
                            System.out.println("执行业务逻辑");
                            break;  // 执行成功，退出循环
                        } finally {
                            lockB.unlock();
                        }
                    }
                } finally {
                    lockA.unlock();
                }
            }
            // 获取失败时，不立即重试，避免活锁
            Thread.sleep(50);
        }
    }
}
```

### Step 6：线程池中的死锁（隐式死锁）

**操作说明：**

线程池中的死锁更隐蔽：所有线程都在等待一个永远不会被执行的任务。

```java
public class ThreadPoolDeadlock {
    static ExecutorService pool = Executors.newSingleThreadExecutor();

    public static void main(String[] args) throws Exception {
        // 外层任务
        Future<String> outerFuture = pool.submit(() -> {
            System.out.println("外层任务开始执行");

            // 内层任务提交到同一个单线程池
            Future<String> innerFuture = pool.submit(() -> {
                System.out.println("内层任务开始执行");
                return "inner result";
            });

            try {
                // 外层等待内层结果 — 但内层需要等外层执行完才能被执行！
                return innerFuture.get();  // ⚠️ 死锁！
            } catch (Exception e) {
                return "error: " + e.getMessage();
            }
        });

        System.out.println(outerFuture.get());
    }
}
```

> 💡 **AI 辅助用法**：解释上面的代码为什么会产生死锁，以及如何修复（提示：换用 newCachedThreadPool？分开不同的线程池？）

---

## 验证标准

完成实践后，逐项检查：

- [ ] 能手动创建带完整参数的 ThreadPoolExecutor
- [ ] 运行性能对比实验后，能解释核心线程数对性能的影响
- [ ] 能复现死锁并解释其成因（四个必要条件）
- [ ] 能使用 jstack 工具排查死锁
- [ ] 知道至少两种解决死锁的编码策略
- [ ] 理解线程池隐式死锁的场景和避免方式

---

## 思考题

1. 死锁的四个必要条件是什么？（互斥、持有并等待、不可剥夺、循环等待）破坏其中哪一个最容易实现？
2. 当线程池的 corePoolSize = 4, maxPoolSize = 8, queue = 10 时，提交第 15 个任务会发生什么？第 20 个呢？
3. 使用 AI 生成一个线程池参数配置不合理的场景（如队列太短导致频繁创建销毁线程），并解释其对 GC 和 CPU 的影响。
4. `CallerRunsPolicy` 为什么可以起到"反压"（backpressure）的作用？它和直接抛出异常相比有什么优势？

---

## 常用排查工具速查

| 工具            | 命令                         | 用途                           |
| ------------- | -------------------------- | ---------------------------- |
| **jps**       | `jps -l`                   | 查看 Java 进程 ID                |
| **jstack**    | `jstack -l <pid>`          | 打印线程堆栈（排查死锁）                 |
| **jstat**     | `jstat -gcutil <pid> 1000` | 每秒查看 GC 情况                   |
| **jvisualvm** | `jvisualvm`                | 可视化监控（线程、CPU、内存）             |
| **JMC**       | `jmc`                      | Java Mission Control（更高级的监控） |

> 💡 **AI 辅助排查套路**：
> 把以下信息发给 AI → 让 AI 分析：
> 
> 1. 完整异常栈（Exception Stack Trace）
> 2. jstack 输出（怀疑死锁时）
> 3. 代码片段 + "这段代码有什么线程安全问题？"

---

*上一节实操：[实操 02.1 售票案例](practice-02.1-ticket-selling-and-concurrency-issues.md) | 下一章：[03-AI Agent入门](../../03-introduction-to-ai-agent/theory/)*
