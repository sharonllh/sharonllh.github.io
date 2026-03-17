---
layout: post
title: .NET中的sync-over-async问题是什么？
date: 2026-03-16 +0800
tags: [.NET, 线程]
categories: [.NET]
---

平时在工作中，总是听到sync-over-async这个词，只知道它是指在一个sync函数中调用async函数并等待结果，可能会导致thread starvation，但是不知道为什么。今天抽空研究了一下这个问题，在这里记录一下。

## 纯Async调用的线程行为

首先从下面的例子入手，理解在纯async调用中，线程是怎么工作的。

```cs
using System;
using System.Diagnostics;
using System.Threading;
using System.Threading.Tasks;

namespace SyncOverAsyncDemo;

class Program
{
    static Stopwatch sw;

    static async Task Main(string[] args)
    {
        Console.WriteLine("=== Pure Async Demo ===\n");

        sw = Stopwatch.StartNew();

        Console.WriteLine($"  Step 1 [{sw.ElapsedMilliseconds,5}ms] [Thread {Thread.CurrentThread.ManagedThreadId}] (Caller thread) Calls DoAsyncWork()");

        string result = await DoAsyncWork();

        Console.WriteLine($"  Step 5 [{sw.ElapsedMilliseconds,5}ms] [Thread {Thread.CurrentThread.ManagedThreadId}] (Thread pool thread, not necessarily same as step 3-4, but often is) Runs continuation of Main, got result: {result}");

        Console.WriteLine("\nDone.");
    }

    static async Task<string> DoAsyncWork()
    {
        Console.WriteLine($"  Step 2 [{sw.ElapsedMilliseconds,5}ms] [Thread {Thread.CurrentThread.ManagedThreadId}] (Caller thread, always same as step 1) Inside DoAsyncWork, starts async operation, caller thread is now FREE, no thread blocked during wait");

        await Task.Delay(2000);

        Console.WriteLine($"  Step 3 [{sw.ElapsedMilliseconds,5}ms] [Thread {Thread.CurrentThread.ManagedThreadId}] (Thread pool thread, NOT necessarily same as step 1-2) Async operation done, runs continuation inside DoAsyncWork");
        Console.WriteLine($"  Step 4 [{sw.ElapsedMilliseconds,5}ms] [Thread {Thread.CurrentThread.ManagedThreadId}] (Thread pool thread, always same as step 3) Completes the Task with result");

        return $"done on thread {Thread.CurrentThread.ManagedThreadId}";
    }
}
```

这个例子的输出为：
```
=== Pure Async Demo ===

  Step 1 [    0ms] [Thread 1] (Caller thread) Calls DoAsyncWork()
  Step 2 [    1ms] [Thread 1] (Caller thread, always same as step 1) Inside DoAsyncWork, starts async operation, caller thread is now FREE, no thread blocked during wait
  Step 3 [ 2013ms] [Thread 9] (Thread pool thread, NOT necessarily same as step 1-2) Async operation done, runs continuation inside DoAsyncWork
  Step 4 [ 2013ms] [Thread 9] (Thread pool thread, always same as step 3) Completes the Task with result
  Step 5 [ 2013ms] [Thread 9] (Thread pool thread, not necessarily same as step 3-4, but often is) Runs continuation of Main, got result: done on thread 9

Done.
```

可以看出，在纯async调用中，异步操作不会阻塞任何线程。
- Caller thread在调用完异步方法后，会被归还到thread pool中。
- 异步操作执行时，不会阻塞任何线程。
- 异步操作完成后，操作系统会发出信号，然后由thread pool中的thread来处理`DoAsyncWork`函数中的后续工作，包括：
    - Run continuation: 运行`await`后面的代码，包括`return`。
    - Complete Task: 设置`Task`的结果和状态，并唤醒等待的线程。
- 继续由thread pool中的thread来处理`Main`函数中的后续工作。
    - 在`Main`函数和`DoAsyncWork`函数中执行后续工作的threadpool thread不一定是同一个，但是一般会是同一个。

## Sync-Over-Async的线程行为

接下来，我们来看一个sync-over-async的例子。

```cs
using System;
using System.Diagnostics;
using System.Threading;
using System.Threading.Tasks;

namespace SyncOverAsyncDemo;

class Program
{
    static Stopwatch sw;

    static void Main(string[] args)
    {
        Console.WriteLine("=== Sync Over Async Demo ===\n");

        sw = Stopwatch.StartNew();

        Console.WriteLine($"  Step 1 [{sw.ElapsedMilliseconds,5}ms] [Thread {Thread.CurrentThread.ManagedThreadId}] (Caller thread) Calls DoAsyncWork() and BLOCKS on .GetResult()");

        string result = DoAsyncWork().GetAwaiter().GetResult();

        Console.WriteLine($"  Step 5 [{sw.ElapsedMilliseconds,5}ms] [Thread {Thread.CurrentThread.ManagedThreadId}] (Caller thread, always same as step 1-2) Unblocked after Task completed, got result: {result}");

        Console.WriteLine("\nDone.");
    }

    static async Task<string> DoAsyncWork()
    {
        Console.WriteLine($"  Step 2 [{sw.ElapsedMilliseconds,5}ms] [Thread {Thread.CurrentThread.ManagedThreadId}] (Caller thread, always same as step 1) Inside DoAsyncWork, starts async operation, caller thread BLOCKED on .GetResult() after this");

        await Task.Delay(2000);

        Console.WriteLine($"  Step 3 [{sw.ElapsedMilliseconds,5}ms] [Thread {Thread.CurrentThread.ManagedThreadId}] (Thread pool thread, always DIFFERENT from step 1-2 because caller thread is blocked) Async operation done, runs continuation inside DoAsyncWork");
        Console.WriteLine($"  Step 4 [{sw.ElapsedMilliseconds,5}ms] [Thread {Thread.CurrentThread.ManagedThreadId}] (Thread pool thread, always same as step 3) Completes the Task, this unblocks the caller thread");

        return $"done on thread {Thread.CurrentThread.ManagedThreadId}";
    }
}
```

这个例子的输出为：
```
=== Sync Over Async Demo ===

  Step 1 [    0ms] [Thread 1] (Caller thread) Calls DoAsyncWork() and BLOCKS on .GetResult()
  Step 2 [    2ms] [Thread 1] (Caller thread, always same as step 1) Inside DoAsyncWork, starts async operation, caller thread BLOCKED on .GetResult() after this
  Step 3 [ 2013ms] [Thread 8] (Thread pool thread, always DIFFERENT from step 1-2 because caller thread is blocked) Async operation done, runs continuation inside DoAsyncWork
  Step 4 [ 2013ms] [Thread 8] (Thread pool thread, always same as step 3) Completes the Task, this unblocks the caller thread   
  Step 5 [ 2013ms] [Thread 1] (Caller thread, always same as step 1-2) Unblocked after Task completed, got result: done on thread 8

Done.
```

可以看出，在sync-over-async调用中，异步操作会阻塞线程。
- Caller thread在调用完异步方法后，会被阻塞，等待异步调用的结果。
- 异步操作执行时，Caller thread一直被阻塞。
- 异步操作完成后，操作系统会发出信号，然后由thread pool中的thread来处理`DoAsyncWork`函数中的后续操作（包括Run continuation和Complete Task）。
- 返回`Main`函数后，Caller thread继续处理异步调用后面的工作。

## Sync-Over-Async导致的Thread Starvation

假设服务器端的线程池中共有1000个线程，同时有1000个请求过来了。

对于纯async调用，1000个异步操作进行时，0个线程在等待，线程池中的1000个线程可以继续处理新请求或者Run continuation & Complete Task。

对于sync-over-async调用，1000个异步操作进行时，1000个线程被阻塞，线程池中没有线程可以处理新请求或者Run continuation & Complete Task。如果没有线程去Run continuation & Complete Task，这1000个线程就会一直等待。线程池耗尽后，会以非常缓慢的速度创建新的线程（每秒1~2个线程），因此服务器并不会彻底陷入死锁状态，只是throughput会变得非常低。这就是所谓的thread starvation。

下面的例子很好地展示了什么是thread starvation：
```cs
using System;
using System.Diagnostics;
using System.Threading;
using System.Threading.Tasks;

namespace SyncOverAsyncDemo;

class Program
{
    static Stopwatch sw;

    static void Main(string[] args)
    {
        ThreadPool.SetMinThreads(4, 4);
        ThreadPool.SetMaxThreads(4, 4);

        Console.WriteLine("=== Sync Over Async — Thread Starvation Demo ===");
        Console.WriteLine("Setup: thread pool limited to 4 threads, launching 4 sync-over-async work items to consume all threads");

        sw = Stopwatch.StartNew();

        var countdown = new CountdownEvent(4);

        for (int i = 0; i < 4; i++)
        {
            int id = i;
            ThreadPool.QueueUserWorkItem(_ =>
            {
                Console.WriteLine($"  [{sw.ElapsedMilliseconds,5}ms] [Thread {Thread.CurrentThread.ManagedThreadId}] Request {id}: starts, calls DoAsyncWork().GetResult() — this thread is now BLOCKED");

                string result = DoAsyncWork(id).GetAwaiter().GetResult();

                Console.WriteLine($"  [{sw.ElapsedMilliseconds,5}ms] [Thread {Thread.CurrentThread.ManagedThreadId}] Request {id}: unblocked, got result: {result}");
                countdown.Signal();
            });
        }

        Console.WriteLine($"[{sw.ElapsedMilliseconds,5}ms] All 4 threads blocked on .GetResult(), each Task needs a thread to complete but none available. Waiting up to 10 seconds...");

        bool completed = countdown.Wait(TimeSpan.FromSeconds(10));

        if (!completed)
        {
            Console.WriteLine($"[{sw.ElapsedMilliseconds,5}ms] *** THREAD STARVATION *** 4 threads blocked, 0 available to complete Tasks. No Task can complete -> no thread can unblock -> stuck.");
        }
        else
        {
            Console.WriteLine($"[{sw.ElapsedMilliseconds,5}ms] All requests completed (thread pool injected new threads in time)");
        }

        Console.WriteLine("Done.");
    }

    static async Task<string> DoAsyncWork(int id)
    {
        await Task.Delay(1000);
        Console.WriteLine($"  [{sw.ElapsedMilliseconds,5}ms] [Thread {Thread.CurrentThread.ManagedThreadId}] Request {id}: async operation finished, needs a thread pool thread to complete the Task — but none available!");
        return $"request {id} done on thread {Thread.CurrentThread.ManagedThreadId}";
    }
}
```

这个例子的输出为：
```
=== Sync Over Async - Thread Starvation Demo ===
Setup: thread pool limited to 4 threads, launching 4 sync-over-async work items to consume all threads
[    0ms] All 4 threads blocked on .GetResult(), each Task needs a thread to complete but none available. Waiting up to 10 seconds...
  [    1ms] [Thread 6] Request 2: starts, calls DoAsyncWork().GetResult() - this thread is now BLOCKED
  [    1ms] [Thread 9] Request 1: starts, calls DoAsyncWork().GetResult() - this thread is now BLOCKED
  [    1ms] [Thread 10] Request 0: starts, calls DoAsyncWork().GetResult() - this thread is now BLOCKED
  [    1ms] [Thread 8] Request 3: starts, calls DoAsyncWork().GetResult() - this thread is now BLOCKED
[10015ms] *** THREAD STARVATION *** 4 threads blocked, 0 available to complete Tasks. No Task can complete -> no thread can unblock -> stuck.
Done.
```

可以看到，明明异步操作1秒就完成了，但是线程池中的4个线程却一直被阻塞，就是因为没有多余的线程去Run continuation & Complete Task。

需要注意的是，这个例子限制了线程池的最大线程数为4，因此永远不会生成新线程。在实际工作中，最大线程数通常设置的比较大，还是会生成新线程的，只是生成的速度非常慢。

## Sync-Over-Async导致的死锁

在一些比较老的.NET Framework中，sync-over-async甚至会导致死锁。

在这些老框架中，有一个`SynchronizationContext`的概念，会导致只能由特定的线程去Run continuation & Complete Task。如果这个特定的线程恰好是那个等待Task的线程，就会导致死锁。

为了解决这个问题，通常会用`ConfigureAwait(false)`来告诉runtime，让线程池中的线程来Run continuation & Complete Task。例如：
```cs
await Task.Delay(1000).ConfigureAwait(false); 
```

需要注意的是，在一个异步函数中使用`ConfigureAwait(false)`，只会影响这个函数的Run continuation & Complete Task由哪个线程来执行。为了彻底避免死锁，我们需要在整个调用链的所有异步函数中都使用`ConfigureAwait(false)`。

这个问题就不过多展开了，毕竟在新的.NET Core中，已经没有`SynchronizationContext`的概念了，也就不会出现因此而导致的死锁问题了。