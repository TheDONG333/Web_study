# 实操 02.1：手写售票 & 库存扣减 — 复现并解决线程安全问题

> ⏱ **预计时长**：60 分钟
> 📌 **难度**：⭐⭐⭐
> 🔧 **AI 辅助**：用 AI 生成并发问题复现代码，辅助排查 Unsafe 代码中的安全隐患

---

## 前置要求

- ✅ 已完成章节：02.1 线程创建与生命周期、02.2 线程安全 + synchronized + Lock
- ✅ 需要安装：JDK 17+、任意 Java IDE（推荐 IntelliJ IDEA）
- ✅ 环境准备：能创建并运行 Java 项目

## 实践目标

完成本次实操后，你将能够：

1. 复现经典的"超卖"线程安全问题
2. 分别用 synchronized、Lock、Atomic 类解决线程安全问题
3. 对比三种方案的性能差异
4. 使用 AI 辅助分析和排查并发 bug

---

## 背景：经典的高铁售票 & 库存扣减

> **场景描述**：
> 
> - 高铁有 100 张票，5 个售票窗口同时卖票
> - 在不加锁的情况下，同一张票可能被多个窗口卖出（超卖）
> - 库存扣减同理：100 件商品，10 个用户同时下单

---

## 步骤

### Step 1：搭建项目环境

**操作说明：**

创建一个 Maven 项目或在 `02-concurrency-programming/code/` 下创建源代码文件。

**项目结构：**

```
code/concurrency-demo/
└── src/
    └── main/java/com/agent/learn/concurrency/
        ├── ticket/           # 售票案例
        │   ├── UnsafeTicketSelling.java
        │   ├── SyncTicketSelling.java
        │   └── LockTicketSelling.java
        ├── inventory/        # 库存扣减案例
        │   ├── UnsafeInventory.java
        │   └── SafeInventory.java
        └── util/
            └── ThreadPoolFactory.java
```

### Step 2：复现线程安全问题 — 售票案例

**操作说明：**

先写一个**不安全的售票代码**，观察超卖现象。

```java
package com.agent.learn.concurrency.ticket;

public class UnsafeTicketSelling {
    // 总票数
    private int tickets = 100;

    // ❌ 未加锁的售票方法
    public void sellTicket(String windowName) {
        if (tickets > 0) {
            // 模拟出票延迟（让问题更容易复现）
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            System.out.println(windowName + " 卖出了第 " + tickets + " 张票");
            tickets--;
        } else {
            System.out.println(windowName + "：票已售罄");
        }
    }

    public static void main(String[] args) {
        UnsafeTicketSelling system = new UnsafeTicketSelling();

        // 5 个窗口同时卖票
        for (int i = 1; i <= 5; i++) {
            final String windowName = "窗口" + i;
            new Thread(() -> {
                // 每个窗口尝试卖 30 次
                for (int j = 0; j < 30; j++) {
                    system.sellTicket(windowName);
                }
            }).start();
        }
    }
}
```

**运行并观察问题：**

- 同一张票被卖出了好几次（超卖）
- 总卖出的票数超过了 100
- 甚至出现了第 0 张、第 -1 张票

> 💡 **AI 辅助用法**：将上面的 UnsafeTicketSelling 代码发给 AI，让 AI 指出所有线程安全问题，并解释为什么 `tickets > 0` 和 `tickets--` 之间存在竞态条件。

### Step 3：用 synchronized 解决

```java
public class SyncTicketSelling {
    private int tickets = 100;
    private final Object lock = new Object();

    public void sellTicket(String windowName) {
        synchronized (lock) {  // 加锁
            if (tickets > 0) {
                try {
                    Thread.sleep(10);  // 保留延迟模拟
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                System.out.println(windowName + " 卖出了第 " + tickets + " 张票");
                tickets--;
            }
        }  // 自动释放锁
    }

    // 或者直接同步方法（更简洁）
    public synchronized void sellTicketV2(String windowName) {
        // 等价于 synchronized(this) { ... }
    }
}
```

### Step 4：用 Lock 解决

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class LockTicketSelling {
    private int tickets = 100;
    private final Lock lock = new ReentrantLock();

    public void sellTicket(String windowName) {
        lock.lock();
        try {
            if (tickets > 0) {
                Thread.sleep(10);
                System.out.println(windowName + " 卖出了第 " + tickets + " 张票");
                tickets--;
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            lock.unlock();  // 必须释放！
        }
    }
}
```

### Step 5：库存扣减案例（加需求）

**场景扩展**：库存扣减 + 记录扣减日志 + 防止重复扣减。

```java
public class SafeInventory {
    // 使用 AtomicInteger 进行原子扣减
    private final AtomicInteger stock = new AtomicInteger(100);
    // 记录已扣减成功的用户
    private final ConcurrentHashMap<String, Boolean> records = new ConcurrentHashMap<>();

    /**
     * 扣减库存（CAS 原子操作 + 防重复）
     * @return true=扣减成功，false=库存不足或重复操作
     */
    public boolean deduct(String userId, int quantity) {
        // 1. 防止重复扣减
        if (records.putIfAbsent(userId, true) != null) {
            System.out.println(userId + " 已扣减过，拒绝重复操作");
            return false;
        }

        // 2. CAS 扣减
        while (true) {
            int currentStock = stock.get();
            if (currentStock < quantity) {
                System.out.println(userId + " 扣减失败：库存不足（剩余 " + currentStock + "）");
                records.remove(userId);  // 回滚记录
                return false;
            }
            if (stock.compareAndSet(currentStock, currentStock - quantity)) {
                System.out.println(userId + " 扣减成功，扣减 " + quantity +
                        "，剩余库存 " + (currentStock - quantity));
                return true;
            }
            // CAS 失败则自旋重试
        }
    }
}
```

### Step 6：性能对比测试

**操作说明：**

写一段测试代码，对比三种方式的性能：

```java
public class PerformanceTest {
    private static final int THREAD_COUNT = 10;
    private static final int OPERATIONS = 10000;

    public static void main(String[] args) throws Exception {
        System.out.println("=== 性能对比测试 ===");
        System.out.println("线程数：" + THREAD_COUNT + "，操作数：" + OPERATIONS);

        // 测试 synchronized
        testSync();
        // 测试 Lock
        testLock();
        // 测试 Atomic
        testAtomic();
    }

    private static long measureTime(Runnable task) {
        long start = System.nanoTime();
        task.run();
        return System.nanoTime() - start;
    }
    // ... 完整实现见 code/ 目录
}
```

> 💡 **AI 辅助用法**：将三种方案的性能测试结果发给 AI，让 AI 分析为什么某些方案在某些场景下更快。

---

## 验证标准

完成实践后，逐项检查：

- [x] Unsafe 版本运行时出现了超卖/负数票数
- [x] synchronized 版本没有超卖且票数总和正确
- [x] Lock 版本没有超卖且票数总和正确
- [x] 能用 AI 解释三种方式各自的工作原理
- [x] 理解 `tickets > 0` 和 `tickets--` 之间为什么会有竞态条件
- [x] 库存扣减案例中使用了 `putIfAbsent` 防止重复扣减

---

## 思考题

1. 如果把 UnsafeTicketSelling 中的 `Thread.sleep(10)` 去掉，还会出现线程安全问题吗？为什么？（提示：指令重排序）
2. synchronized 和 Lock 在你本机的性能对比结果如何？为什么？
3. 用 AI 生成一段会死锁的代码，然后分析死锁的成因和解决方式。
4. 在库存扣减案例中，`stock.decrementAndGet()` 和 `stock.compareAndSet()` 实现方式的区别是什么？

---

## 常见问题

**Q：为什么打印的票号是乱的？不是应该按顺序递减吗？**
A：多个线程交替执行，而且 `System.out.println` 本身不是原子操作。如果需要按顺序打印，可以把打印也纳入同步块。

**Q：tickets-- 分解成了几步？**
A：三步：① 读取 tickets 的值 ② 值减 1 ③ 写回 tickets。三步之间可能被其他线程中断。

---

*上一节理论：[02.2 线程安全 + synchronized + Lock](../theory/02.2-thread-safety-synchronized-lock.md) | 下一节实操：[实操 02.2 线程池调优与死锁分析](practice-02.2-threadpool-tuning-and-deadlock-analysis.md)*
