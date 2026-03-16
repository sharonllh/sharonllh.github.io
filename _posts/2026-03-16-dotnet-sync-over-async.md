---
layout: post
title: .NET中的sync-over-async问题是什么？
date: 2026-03-16 +0800
tags: [.NET, 线程]
categories: [.NET]
---

平时在工作中，总是听到sync-over-async这个词，只知道它是指在一个sync函数中调用async函数并等待结果，可能会导致thread starvation，但是不知道为什么。今天抽空研究了一下这个问题，在这里记录一下。

## Pure Async

首先从下面的例子入手，理解在pure async的情况下，线程是怎么工作的。

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

可以看出，在pure async的情况下，异步操作不会阻塞任何线程。
- Caller thread在调用完异步方法后，会被归还到thread pool中。
- 异步操作执行时，不会阻塞任何线程。
- 异步操作完成后，操作系统会发出信号，然后由thread pool中的thread来处理后面的工作。
- 在`Main`函数和`DoAsyncWork`函数中执行后续工作的threadpool thread不一定是同一个，但是一般会是同一个。

## Sync-Over-Async

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


