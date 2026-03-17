---
layout: post
title: 在.NET中，把同步操作放入Task.Run会发生什么？
date: 2026-03-17 +0800
tags: [.NET, 线程，异步]
categories: [.NET]
---

大家应该都知道，把一个同步操作放入`Task.Run`，可以把同步变为异步。然而实际工作中，大家并不会这么做，还会称这种方法为“伪异步”。这是为什么呢？这里我们来探讨一下。

首先，我们从一个例子入手，看看把一个同步操作放入`Task.Run`会发生什么。

```cs
using System.Diagnostics;

namespace TaskRunDemo;

class Program
{
    static Stopwatch sw;

    static async Task Main(string[] args)
    {
        sw = Stopwatch.StartNew();

        Console.WriteLine("=== Task.Run Wrapping Sync Operation — Thread Behavior Demo ===");
        Console.WriteLine();

        await WrappedInTaskRun();

        Console.WriteLine();
        Console.WriteLine("Done.");
    }

    static async Task WrappedInTaskRun()
    {
        int callerThread = Thread.CurrentThread.ManagedThreadId;
        Console.WriteLine($"  [{sw.ElapsedMilliseconds,5}ms] [Thread {callerThread}] Step 1: Calling Task.Run(() => SyncOperation()) — thread {callerThread} is FREE (returned to thread pool)");

        string result = await Task.Run(() =>
        {
            int workerThread = Thread.CurrentThread.ManagedThreadId;
            Console.WriteLine($"  [{sw.ElapsedMilliseconds,5}ms] [Thread {workerThread}] (could be any thread pool thread including caller, but typically not) Step 2: A thread pool thread picked up the work — thread {workerThread} is now BLOCKED during the operation");
            string r = SyncOperation();
            Console.WriteLine($"  [{sw.ElapsedMilliseconds,5}ms] [Thread {Thread.CurrentThread.ManagedThreadId}] (always same as step 2) Step 3: Sync operation completed, Task is completed");
            return r;
        });

        Console.WriteLine($"  [{sw.ElapsedMilliseconds,5}ms] [Thread {Thread.CurrentThread.ManagedThreadId}] (often same as step 3 as runtime optimization, but not guaranteed) Step 4: Continuation runs — got result: {result}");
    }

    static string SyncOperation()
    {
        Thread.Sleep(2000);
        return $"done on thread {Thread.CurrentThread.ManagedThreadId}";
    }
}
```

这个例子的输出如下：
```
=== Task.Run Wrapping Sync Operation - Thread Behavior Demo ===

  [    4ms] [Thread 1] Step 1: Calling Task.Run(() => SyncOperation()) - thread 1 is FREE (returned to thread pool)
  [    6ms] [Thread 9] (could be any thread pool thread including caller, but typically not) Step 2: A thread pool thread picked up the work - thread 9 is now BLOCKED during the operation
  [ 2019ms] [Thread 9] (always same as step 2) Step 3: Sync operation completed, Task is completed
  [ 2020ms] [Thread 9] (often same as step 3 as runtime optimization, but not guaranteed) Step 4: Continuation runs - got result: done on thread 9

Done.
```

步骤2的线程可能是线程池中的任意线程，包括Caller thread，但通常不是Caller thread，这是因为：
- 在Caller thread调用`Task.Run`之后，`Task.Run`包含的工作会被放入线程池的队列中，等待一个线程取出并处理它。
- 在Caller thread执行到`await`时，Caller thread被归还到线程池中。
- 如果线程池中有空闲的线程，很可能在Caller thread被归还之前，就已经把队列中的工作取出了。

步骤4的线程通常和步骤2-3的一样，这是因为runtime的inline optimization。
- 在线程complete Task时，runtime会检查是否有continuation在等待执行，如果有，就直接在当前线程上继续执行，而不是把continuation放入线程池的队列中。
- 这样做可以避免排队和context switch，因此执行会更快。
- Inline optimization只是一个内部优化，并不是一个保证会发生的feature。

可以看出，把一个同步操作放入`Task.Run`之后，只是把线程阻塞从Caller thread转移到另一个线程池的线程上了，并没有减少线程阻塞。

这样做甚至不如直接调用同步操作，因为会带来额外的开销。
- 排队的开销：同步操作会被放到线程池的队列中。
- Context switch的开销：从Caller thread切换到另一个线程执行，需要进行context switch。
- 内存和GC的开销：需要在堆上创建Task对象，可能会导致更频繁的GC。

所以啊，如果要调用一个同步操作，直接调用就行了，不要把它放到`Task.Run`里。