# synchronized

## 面试常见问法

- `synchronized` 的作用是什么？
- 它可以锁哪些对象？
- `synchronized` 和 `ReentrantLock` 有什么区别？
- `synchronized` 能解决分布式并发问题吗？

## 核心回答

`synchronized` 是 Java 内置锁，用于保证同一时刻只有一个线程进入同一把锁保护的临界区。它可以修饰实例方法、静态方法和代码块，锁对象分别是当前实例、类对象或指定对象。它能保证互斥和可见性，适合保护 JVM 内共享变量，但不能解决多实例部署下的分布式并发问题。

## 背景

在 Java 后端开发中，共享资源的并发访问是最常见的线程安全问题。库存扣减、计数器自增、单例初始化、缓存加载等场景都需要互斥保护。`synchronized` 是 Java 语言层面的内置同步机制，使用最广泛、学习成本最低。面试中它通常作为并发基础的第一个问题，面试官会从用法切入，追问到 JVM 层面的 Monitor、对象头 Mark Word 和锁升级机制。

## 核心概念

面试时要把锁对象讲清楚：锁住代码不是目的，本质是多个线程竞争同一个监视器对象。

```mermaid
flowchart LR
    A["无锁状态"] --> B["偏向锁"]
    B --> C["轻量级锁"]
    C --> D["重量级锁"]
    
    B -- "另一个线程尝试获取" --> C
    C -- "自旋失败" --> D
```

> JDK 6 引入了锁升级优化。偏向锁适合只有一个线程访问的场景，几乎无开销；轻量级锁通过 CAS 自旋避免阻塞，适合竞争短暂的场景；重量级锁会将竞争失败的线程挂起，涉及操作系统层面的线程切换。锁只能升级不能降级。

## 项目结合

在单 JVM 内维护本地计数器时，可以使用 `synchronized` 保证自增操作原子性。

```java
public class LocalCounter {
    private long count;

    public synchronized long incrementAndGet() {
        // 自增不是原子操作，需要加锁保护共享变量
        count++;
        return count;
    }
}
```

使用代码块可以缩小锁范围。

```java
public class InventoryService {
    private final Object lock = new Object();
    private int stock = 100;

    public boolean deduct() {
        synchronized (lock) {
            // 只保护库存扣减这段临界区，减少锁范围
            if (stock <= 0) {
                return false;
            }
            stock--;
            return true;
        }
    }
}
```

## 深入追问

- **实例方法锁什么？** 锁当前对象 `this`。两个不同的实例调用同一个 synchronized 实例方法不会互斥，因为锁对象不同。

- **静态方法锁什么？** 锁当前类的 `Class` 对象（如 `InventoryService.class`）。同一个类的所有实例共享同一把锁。

- **对象头 Mark Word 的结构是什么？** 在 64 位 JVM 中，Mark Word 占 64 位。无锁状态存储对象的 hashCode、GC 分代年龄；偏向锁状态存储偏向线程 ID；轻量级锁状态存储指向栈帧中锁记录（Lock Record）的指针；重量级锁状态存储指向 Monitor 对象的指针。JVM 通过 Mark Word 最后几位的标志位来区分当前锁状态。

- **锁升级过程是怎样的？** 初始状态为无锁或偏向锁。当第一个线程获取锁时，JVM 通过 CAS 将线程 ID 写入 Mark Word，变为偏向锁。当第二个线程尝试获取时，偏向锁撤销，升级为轻量级锁，竞争线程通过 CAS 自旋尝试获取。如果自旋达到阈值仍未成功，升级为重量级锁，竞争失败的线程进入 Monitor 的等待队列被挂起。锁只能升级不能降级（偏向锁在 JDK 15 之后默认关闭）。

- **为什么单机锁不能解决分布式问题？** 因为不同 JVM 中锁对象不共享，`synchronized` 只在同一个 JVM 进程内有效。分布式场景需要 Redis 分布式锁、ZooKeeper 或数据库行锁。

- **`synchronized` 是可重入的吗？** 是的。同一个线程可以重复获取同一把锁，JVM 通过 Monitor 的计数器记录重入次数，每次退出临界区计数减一，归零时释放锁。

## 常见问题

- **不要把锁对象暴露给外部代码**：如果外部代码拿到了锁对象并对其加锁，可能导致意料之外的死锁或性能问题。应使用 `private final Object lock` 作为内部锁。
- **不要锁字符串常量或包装类型的缓存对象**：`"abc"` 这样的字符串常量在常量池中共享，`Integer.valueOf(1)` 会复用缓存对象。不同代码锁同一个对象会导致不相关的逻辑互相阻塞。
- **不要扩大锁范围**：把 IO 操作、RPC 调用等耗时逻辑放在 `synchronized` 块内会严重降低并发度。应只保护最小的共享变量操作。
- **不要忽略 `wait/notify` 的使用规范**：`wait` 必须在 `synchronized` 块内调用，且应放在 `while` 循环中检查条件（防止虚假唤醒）。`notifyAll` 通常比 `notify` 更安全。

## 线上案例

**锁对象不一致导致库存超卖**：某电商系统的库存扣减方法使用 `synchronized(this)` 加锁。但该 Service 类未配置为单例（Spring 中 prototype 作用域），每次请求注入的是不同的实例，`this` 指向不同对象，导致多个请求实际上没有竞争同一把锁。高并发下库存出现负数。排查方式是通过日志打印锁对象的 `System.identityHashCode`，发现每次请求的值不同。修复方案是将锁对象改为 `private static final Object LOCK`（类级别共享），或确认 Service 是 Spring 单例。

## 相关笔记

- [ReentrantLock](./08-ReentrantLock.md)：`synchronized` 的显式锁替代方案，提供超时、可中断和条件队列。
- [volatile](./09-volatile.md)：`synchronized` 保证互斥和可见性，`volatile` 只保证可见性和有序性。
- [AQS](./12-AQS.md)：`ReentrantLock` 底层基于 AQS，理解 AQS 可以更好地对比两种锁机制。

## 一句话总结

`synchronized` 是 JVM 内置互斥锁，面试重点是锁对象（实例/类/指定对象）、Mark Word 与锁升级（偏向→轻量级→重量级）、可见性保证和单机边界。
