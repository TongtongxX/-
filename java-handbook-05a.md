# Java 语法学习手册（面向 Go 开发者）

## 第 5 部分（上）：并发基础

> 本文件包含第 29–31 章：`Thread` 与 `Runnable`、线程池 `ExecutorService`、`synchronized` / `volatile` / `Atomic*` / `ReentrantLock`。
> 第 32–34 章（并发容器与 `BlockingQueue`、`CompletableFuture`、虚拟线程）在下一个文件。

**使用约定（同前）**：每个代码块第一行注释是文件名，保存为同名文件后执行 `java 文件名.java` 即可（JDK 21）。带 `main` 的类写在文件最前面，辅助类写在后面。

**先说结论：Java 并发和 Go 的哲学不同。** Go 提倡"通过通信来共享内存"（channel）；Java 传统上是"共享内存 + 锁 + 线程池"，用 `BlockingQueue`（第 32 章）才能得到类似 channel 的写法。Java 21 的虚拟线程（第 34 章）让"一个任务一个线程"的写法重新变得可行，更接近 goroutine。

---

## 29. Thread 与 Runnable

**和 Go 的差异**

| | goroutine | Java 平台线程 |
|---|---|---|
| 创建 | `go f()` | `new Thread(runnable).start()` |
| 成本 | 起步栈很小，可以开几十万个 | 对应一个操作系统线程，64 位 Linux 上默认栈 1MB，开几千个就吃力 |
| 等待结束 | `sync.WaitGroup` | `join()` / `CountDownLatch` / 线程池的 `Future` |
| 取消 | `context.Context` | `interrupt()`，**协作式**，线程自己要响应 |
| 返回结果 | channel | `Callable` + `Future`（第 30 章） |
| 未捕获的 panic / 异常 | **整个进程崩溃** | **只有该线程终止**，其他线程继续运行 |
| 程序何时退出 | `main` 返回就退出，不管 goroutine | **所有非守护线程**结束才退出 |

- `Runnable` 是函数式接口（`void run()`），可以直接用 lambda。
- **`start()` 才会开新线程；直接调用 `run()` 只是普通方法调用**，在当前线程里同步执行。
- `join()` ≈ 等待某个线程结束。`Thread.sleep(ms)` 会抛 checked 的 `InterruptedException`。
- 经典 `for` 的循环变量 `i` 不是"有效 final"，不能直接被 lambda 捕获，要先拷贝一份：`final int id = i;`（增强 for 的变量可以直接捕获）。
- **守护线程**（`setDaemon(true)`）不会阻止 JVM 退出；普通线程会。忘了结束的普通线程会让程序"卡住不退出"。
- **中断**：`thread.interrupt()` 只是设置一个标志，并让阻塞中的 `sleep` / `wait` / `join` 抛出 `InterruptedException`。线程收到后应该收尾退出；如果选择不退出，要用 `Thread.currentThread().interrupt()` 把标志恢复回去，不要悄悄吞掉。
- 线程状态可以用 `getState()` 查看：`NEW`、`RUNNABLE`、`BLOCKED`、`WAITING`、`TIMED_WAITING`、`TERMINATED`。
- 直接用 `new Thread` 在真实项目里很少见，通常用线程池（第 30 章）或虚拟线程（第 34 章）。

```java
// ThreadDemo.java
import java.util.Arrays;

public class ThreadDemo {
    public static void main(String[] args) throws InterruptedException {   // 为了简洁, main 直接声明 throws
        // 1. 创建、启动、等待 ≈ go func() {...}() + wg.Wait()
        Thread t = new Thread(() -> System.out.println("hello from " + Thread.currentThread().getName()), "worker-1");
        System.out.println(t.getState());
        t.start();
        t.join();                                      // 等这个线程结束
        System.out.println(t.getState());

        // 2. 坑: 直接调用 run() 只是普通方法调用, 不会开新线程
        Thread t2 = new Thread(() -> System.out.println("run in " + Thread.currentThread().getName()), "worker-2");
        t2.run();                                      // 在 main 线程里执行
        t2.start();                                    // 在 worker-2 线程里执行
        t2.join();

        // 3. 多个线程, 各自写数组里不同的位置, 最后 join 全部
        int[] results = new int[4];
        Thread[] threads = new Thread[4];
        for (int i = 0; i < threads.length; i++) {
            final int id = i;                          // 经典 for 的 i 不是 effectively final, 要拷贝一份才能被捕获
            threads[i] = new Thread(() -> results[id] = id * id);
            threads[i].start();
        }
        for (Thread th : threads) th.join();           // join 之后, 线程里写入的数据一定可见
        System.out.println(Arrays.toString(results));

        // 4. 线程里的未捕获异常只会终止该线程 (Go 里 goroutine 的 panic 会让整个进程崩溃)
        Thread bad = new Thread(() -> { throw new IllegalStateException("worker crashed"); }, "bad-worker");
        bad.setUncaughtExceptionHandler((thr, ex) -> System.out.println(thr.getName() + " died: " + ex.getMessage()));
        bad.start();
        bad.join();
        System.out.println("main is still alive");

        // 5. 中断: 协作式取消 ≈ context.Cancel
        Thread sleeper = new Thread(() -> {
            try {
                Thread.sleep(10_000);
                System.out.println("woke up normally");
            } catch (InterruptedException e) {
                // 这里选择直接退出; 如果要继续往下跑, 应该先 Thread.currentThread().interrupt() 恢复标志
                System.out.println("interrupted, cleaning up");
            }
        });
        sleeper.start();
        sleeper.interrupt();
        sleeper.join();

        // 6. 程序退出规则: 守护线程不会阻止 JVM 退出, 普通线程会
        Thread daemon = new Thread(() -> {
            try {
                Thread.sleep(5_000);
                System.out.println("daemon finished (never printed)");
            } catch (InterruptedException e) {
                // 示例里为了简洁而忽略
            }
        });
        daemon.setDaemon(true);
        daemon.start();

        new Thread(() -> {
            try {
                Thread.sleep(200);
            } catch (InterruptedException e) {
                // 示例里为了简洁而忽略
            }
            System.out.println("non-daemon worker finished (JVM waited for it)");
        }).start();

        System.out.println("main finished");           // main 先结束, 但 JVM 还会等普通线程
    }
}
```

**运行结果：**

```text
NEW
hello from worker-1
TERMINATED
run in main
run in worker-2
[0, 1, 4, 9]
bad-worker died: worker crashed
main is still alive
interrupted, cleaning up
main finished
non-daemon worker finished (JVM waited for it)
```


## 30. 线程池 ExecutorService

**和 Go 的差异**

- Go 里 goroutine 很便宜，想并发就 `go`，需要限流时自己写 worker pool（N 个 goroutine + channel）。Java 的平台线程很贵，所以几乎总是用**线程池**复用线程，池里自带任务队列。
- 创建：`Executors.newFixedThreadPool(n)`（固定大小）、`newCachedThreadPool()`（按需增长）、`newSingleThreadExecutor()`、`newScheduledThreadPool(n)`（定时任务）、`newVirtualThreadPerTaskExecutor()`（Java 21，第 34 章）。
- 提交任务：
  - `execute(Runnable)`：没有返回值。
  - `submit(Callable<T>)` / `submit(Runnable)`：返回 `Future<T>`。`Callable` 有返回值，还可以抛 checked 异常。
- `Future.get()` **阻塞**直到任务完成，≈ 从 channel 里读结果。可以带超时：`get(100, TimeUnit.MILLISECONDS)`；还有 `cancel(true)`、`isDone()`。
- **任务里的异常陷阱**：`submit` 会把任务抛出的异常存进 `Future`，只有调用 `get()` 时才抛出（包装成 `ExecutionException`，真正的异常在 `getCause()`）。**如果你不调用 `get()`，异常就悄悄消失了。**
- 批量提交：`invokeAll(tasks)` 等全部完成后按提交顺序返回 `Future` 列表。
- **必须关闭线程池**：池里的线程是普通线程，不关闭会让程序无法退出。
  - `shutdown()`：不再接受新任务，等已提交的任务跑完。
  - `shutdownNow()`：尝试中断正在执行的任务。
  - **Java 19+ 的 `ExecutorService` 实现了 `AutoCloseable`**：`try (var pool = ...) { ... }` 结束时自动 `shutdown` 并等待任务完成。推荐这种写法。
- 池大小：CPU 密集型任务大致等于核数；IO 密集型任务可以更大，或者改用虚拟线程。
- 定时任务：`ScheduledExecutorService.schedule(...)` / `scheduleAtFixedRate(...)` ≈ `time.AfterFunc` / `time.Ticker`。

```java
// ExecutorDemo.java
import java.util.*;
import java.util.concurrent.*;

public class ExecutorDemo {
    public static void main(String[] args) throws Exception {
        submitAndGet();
        failuresAndTimeouts();
        reuseAndSchedule();
    }

    static void submitAndGet() throws Exception {
        // 线程池放进 try-with-resources (Java 19+): 结束时自动 shutdown 并等待任务完成
        try (ExecutorService pool = Executors.newFixedThreadPool(4)) {
            // 1. submit(Callable) -> Future: 取回结果 ≈ goroutine + channel
            Future<Integer> f = pool.submit(() -> {
                Thread.sleep(50);
                return 6 * 7;
            });
            System.out.println("computing...");
            System.out.println("result = " + f.get());          // 阻塞直到完成

            // 2. 批量提交, invokeAll 等待全部完成, 按提交顺序返回
            List<Callable<String>> jobs = new ArrayList<>();
            for (int i = 1; i <= 5; i++) {
                final int id = i;
                jobs.add(() -> "job" + id + "=" + id * id);
            }
            List<String> out = new ArrayList<>();
            for (Future<String> r : pool.invokeAll(jobs)) {
                out.add(r.get());
            }
            System.out.println(out);
        }
    }

    static void failuresAndTimeouts() throws Exception {
        try (ExecutorService pool = Executors.newFixedThreadPool(2)) {
            // 3. 任务里的异常: submit 会把它存进 Future, 调用 get() 时才抛出
            //    如果不调用 get(), 这个异常就悄悄消失了
            Runnable failing = () -> { throw new IllegalStateException("task failed"); };
            Future<?> bad = pool.submit(failing);
            try {
                bad.get();
            } catch (ExecutionException e) {
                System.out.println("ExecutionException, cause: " + e.getCause());
            }

            // 4. 超时与取消
            Future<String> slow = pool.submit(() -> {
                Thread.sleep(2_000);
                return "slow";
            });
            try {
                slow.get(100, TimeUnit.MILLISECONDS);
            } catch (TimeoutException e) {
                System.out.println("timeout, cancelling");
                slow.cancel(true);                              // true: 中断正在执行的线程
            }
            System.out.println("cancelled = " + slow.isCancelled());
        }
    }

    static void reuseAndSchedule() throws Exception {
        // 5. 池里的线程会被复用: 20 个任务, 池里只有 2 个线程
        Set<String> names = Collections.synchronizedSet(new TreeSet<>());
        try (ExecutorService small = Executors.newFixedThreadPool(2)) {
            for (int i = 0; i < 20; i++) {
                small.submit(() -> names.add(Thread.currentThread().getName()));
            }
        }                                                       // close(): 等 20 个任务全部执行完
        System.out.println("distinct threads <= 2: " + (names.size() <= 2));

        // 6. 定时任务 ≈ time.AfterFunc / time.Ticker
        try (ScheduledExecutorService sched = Executors.newScheduledThreadPool(1)) {
            ScheduledFuture<String> later = sched.schedule(() -> "fired after delay", 100, TimeUnit.MILLISECONDS);
            System.out.println(later.get());
        }
    }
}
```

**运行结果：**

```text
computing...
result = 42
[job1=1, job2=4, job3=9, job4=16, job5=25]
ExecutionException, cause: java.lang.IllegalStateException: task failed
timeout, cancelling
cancelled = true
distinct threads <= 2: true
fired after delay
```


## 31. synchronized / volatile / Atomic* / ReentrantLock

**和 Go 的差异**

- 数据竞争在 Java 里和 Go 里一样存在，**`count++` 不是原子操作**（读、改、写三步），多线程同时执行会丢失更新。标准 JDK 没有内置类似 Go `-race` 的检测器。下面的例子为了让竞争在任何机器上都稳定复现（真实的 `count++` 窗口很小，单核机器上很难触发），把 `count++` 拆成了"读"和"写"两步，中间插入 `Thread.yield()`。
- 并发要同时解决三件事：**原子性**（复合操作不被打断）、**可见性**（一个线程的写入别的线程能看到）、**有序性**（不被重排）。
- 各个工具能解决什么：

| 工具 | 原子性 | 可见性 | 说明 |
|---|---|---|---|
| `synchronized` | ✓ | ✓ | 对象监视器锁，**自动释放**（包括抛异常时），可重入 |
| `ReentrantLock` | ✓ | ✓ | 手动 `lock()` / `unlock()`，必须放进 `finally`；多了 `tryLock`、可中断、超时、公平锁、多个 `Condition` |
| `volatile` | ✗ | ✓ | 只保证可见性和有序性，**不能让 `i++` 变原子**，适合"状态标志" |
| `AtomicInteger` 等 | ✓ | ✓ | 基于 CAS，无锁；高竞争的计数用 `LongAdder` 更快 |

- **`synchronized`** 有两种写法：
  - 方法上：`synchronized void inc()`，锁的是 `this`（静态方法锁的是类对象）。
  - 代码块：`synchronized (lock) { ... }`，锁的是你指定的对象。
  - 锁对象应该是 `private final Object lock = new Object();`，**不要用字符串、包装类型或对外暴露的对象当锁**。
- **Java 的锁是可重入的**（同一线程可以重复获取），Go 的 `sync.Mutex` 不可重入。
- 对照：

| Go | Java |
|---|---|
| `mu.Lock(); defer mu.Unlock()` | `synchronized` 块；或 `lock.lock(); try { ... } finally { lock.unlock(); }` |
| `sync.RWMutex` | `ReentrantReadWriteLock` |
| `sync/atomic` | `AtomicInteger` / `AtomicLong` / `AtomicReference` / `LongAdder` |
| `sync.WaitGroup` | `CountDownLatch`（一次性）/ `CyclicBarrier` / 线程池 `close()` |
| `sync.Cond` | `Condition`（`lock.newCondition()`） |
| `sync.Map` | `ConcurrentHashMap`（第 32 章） |

- 多个锁要**按固定顺序**获取，否则会死锁；`tryLock(timeout)` 可以避免无限等待。
- `join()`、`synchronized`、`volatile` 都会建立"先行发生"关系，保证之前的写入对之后的读取可见。

```java
// SyncDemo.java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.LongAdder;
import java.util.concurrent.locks.ReentrantLock;

public class SyncDemo {
    static final int THREADS = 4, PER_THREAD = 20_000;

    static int racy = 0;                          // 普通变量: 有数据竞争
    static volatile int volatileCount = 0;        // volatile 也不能让 ++ 变原子
    static int synced = 0;
    static final Object lock = new Object();      // 专用的锁对象
    static int locked = 0;
    static final ReentrantLock rl = new ReentrantLock();
    static final AtomicInteger atomic = new AtomicInteger();
    static final LongAdder adder = new LongAdder();

    // 启动 THREADS 个线程, 每个线程执行 task PER_THREAD 次, 等全部结束
    static void runConcurrently(Runnable task) throws InterruptedException {
        Thread[] ts = new Thread[THREADS];
        for (int i = 0; i < THREADS; i++) {
            ts[i] = new Thread(() -> {
                for (int j = 0; j < PER_THREAD; j++) task.run();
            });
            ts[i].start();
        }
        for (Thread t : ts) t.join();
    }

    public static void main(String[] args) throws Exception {
        counters();
        System.out.println("---");
        volatileFlag();
        System.out.println("---");
        atomicApi();
        lockFeatures();
        latch();
    }

    static void counters() throws InterruptedException {
        int expected = THREADS * PER_THREAD;
        System.out.println("expected          = " + expected);

        // 数据竞争: 为了让它在任何机器上都能稳定复现, 这里把 count++ 的"读-改-写"拆开, 中间让出 CPU
        // (真实的 count++ 在多核机器上同样会丢失更新, 只是窗口很小, 不一定每次都触发)
        // 下面两行的具体数值因机器和运行而异, 但几乎一定远小于 expected
        runConcurrently(() -> {
            int tmp = racy;                           // 读
            Thread.yield();
            racy = tmp + 1;                           // 写: 期间别的线程可能已经改过 racy, 这次写入会覆盖它
        });
        System.out.println("plain int         = " + racy);
        runConcurrently(() -> {
            int tmp = volatileCount;                  // volatile 保证读到最新值, 但"读-改-写"整体仍然可能被打断
            Thread.yield();
            volatileCount = tmp + 1;
        });
        System.out.println("volatile int      = " + volatileCount);

        // 正确做法: 下面这些结果每次都等于 expected
        runConcurrently(() -> {
            synchronized (lock) { synced++; }         // synchronized 块: 自动释放
        });
        System.out.println("synchronized      = " + synced);

        runConcurrently(() -> {
            rl.lock();
            try {
                locked++;
            } finally {
                rl.unlock();                          // 必须放进 finally
            }
        });
        System.out.println("ReentrantLock     = " + locked);

        runConcurrently(atomic::incrementAndGet);
        System.out.println("AtomicInteger     = " + atomic.get());

        runConcurrently(adder::increment);
        System.out.println("LongAdder         = " + adder.sum());

        SafeCounter sc = new SafeCounter();
        runConcurrently(sc::inc);
        System.out.println("synchronized 方法 = " + sc.get());
    }

    // volatile 的正确用法: 状态标志
    static volatile boolean running = true;

    static void volatileFlag() throws InterruptedException {
        Thread worker = new Thread(() -> {
            long spins = 0;
            while (running) spins++;                  // 没有 volatile 的话, 这个循环可能永远看不到 false
            System.out.println("worker saw running=false and stopped");
        });
        worker.start();
        Thread.sleep(100);
        running = false;
        worker.join();
    }

    static void atomicApi() {
        AtomicInteger a = new AtomicInteger(5);
        System.out.println(a.getAndIncrement() + " " + a.incrementAndGet() + " "
                + a.compareAndSet(7, 100) + " " + a.updateAndGet(v -> v * 2));
    }

    static void lockFeatures() throws InterruptedException {
        ReentrantLock l = new ReentrantLock();
        l.lock();
        l.lock();                                     // 可重入: 同一线程可以重复获取
        System.out.println("hold count = " + l.getHoldCount());
        Thread other = new Thread(() -> System.out.println("other thread tryLock = " + l.tryLock()));
        other.start();                                // 别的线程拿不到锁, tryLock 立刻返回 false
        other.join();
        l.unlock();
        l.unlock();
    }

    // CountDownLatch ≈ sync.WaitGroup
    static void latch() throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(3);          // ≈ wg.Add(3)
        AtomicInteger sum = new AtomicInteger();
        for (int i = 1; i <= 3; i++) {
            final int n = i;
            new Thread(() -> {
                sum.addAndGet(n);
                latch.countDown();                             // ≈ wg.Done()
            }).start();
        }
        latch.await();                                         // ≈ wg.Wait()
        System.out.println("sum = " + sum.get());
    }
}

class SafeCounter {
    private int n;

    synchronized void inc() { n++; }              // 锁的是 this
    synchronized int get() { return n; }
}
```

**运行结果：**

```text
expected          = 80000
plain int         = 20000
volatile int      = 20002
synchronized      = 80000
ReentrantLock     = 80000
AtomicInteger     = 80000
LongAdder         = 80000
synchronized 方法 = 80000
---
worker saw running=false and stopped
---
5 7 true 200
hold count = 2
other thread tryLock = false
sum = 6
```


---

## 第 5 部分（上）小结

1. 平台线程 = 操作系统线程，很重；`start()` 才开新线程；未捕获异常只杀死该线程；**所有非守护线程结束 JVM 才退出**。
2. 取消靠 `interrupt()`，是协作式的；收到 `InterruptedException` 要么退出，要么恢复中断标志，别吞掉。
3. 线程池几乎总是首选：`submit` 返回 `Future`；**不调用 `get()` 就看不到任务里的异常**；Java 19+ 用 try-with-resources 自动关闭，否则程序退不出。
4. `count++` 不是原子的：`volatile` 只保证可见性，要原子性用 `synchronized`、`ReentrantLock` 或 `Atomic*`；锁用 `finally` 释放。
5. `CountDownLatch` ≈ `WaitGroup`，`Condition` ≈ `sync.Cond`，Java 的锁可重入。

下一个文件（第 5 部分下）：第 32 章并发容器与 `BlockingQueue`（最接近 channel 的写法）、第 33 章 `CompletableFuture`、第 34 章虚拟线程（最接近 goroutine 的东西）。
