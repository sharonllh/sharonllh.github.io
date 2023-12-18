---
layout: post
title: .NET的TaskCompletionSource类有什么用？
date: 2023-12-18 +0800
tags: [.NET, TaskCompletionSource，异步编程]
categories: [.NET]
---

最近在看同事代码的时候，看到了一个之前从来没用过的类`TaskCompletionSource`，研究了一下这个类是干嘛的，在此记录一下。

## TaskCompletionSource类的作用

简单来说，`TaskCompletionSource`类提供了一种手动控制`Task`完成状态的方式。

回想一下我们平时是怎么使用`Task`的。在创建`Task`时，必须提供一个delegate，表示需要执行什么操作。然后，这个操作会被自动执行，直到完成后，`Task`会被自动设置上相应的完成状态，如`RanToCompletion`, `Faulted`或者`Canceled`。

我们可以创建一个什么都不干的`Task`，然后根据情况灵活地设置它的完成状态吗？用`Task`类是无法实现的，但是用`TaskCompletionSource`类可以。下面是一个简单的例子。

在创建`TaskCompletionSource`时，内部会创建一个什么都不干的`Task`。生产者可以通过`TaskCompletionSource`的`SetResult`等方法设置`Task`的完成状态，而消费者可以在生产者设置完`Task`的完成状态后获取结果。

```cs
TaskCompletionSource<int> tcs = new TaskCompletionSource<int>();
Task<int> task = tcs.Task;

// Start a background task that will complete tcs.Task
Task.Run(() =>
{
    Thread.Sleep(1000);
    tcs.SetResult(15);
});

// The attempt to get the result of task blocks the current thread until the completion source gets signaled
Stopwatch sw = Stopwatch.StartNew();
int result = task.Result;
sw.Stop();

// ElapsedTime should be ~1000ms, task.Result should be 15
Console.WriteLine("(ElapsedTime={0}): task.Result={1}", sw.ElapsedMilliseconds, result);
```

在`TaskCompletionSource`中，有六个方法可以用来设置`Task`的完成状态：
- `SetResult`和`TrySetResult`：设置为`RanToCompletion`状态
- `SetException`和`TrySetException`：设置为`Faulted`状态
- `SetCanceled`和`TrySetCanceled`：设置为`Canceled`状态

`SetXXX`和`TrySetXXX`的区别在于，重复调用`SetXXX`会抛异常，但重复调用`TrySetXXX`不会。

`TaskCompletionSource`类中的所有成员都是线程安全的。

## 源码解析

`TaskCompletionSource`的源码比较简单，主要功能是通过调用`Task`的internal方法实现的。

#### 1. 创建TaskCompletionSource时，内部会创建一个啥都不干的Task

```cs
private readonly Task<TResult> _task;

public TaskCompletionSource() 
    => _task = new Task<TResult>();
```

#### 2. 可以通过属性获取该内部Task

```cs
public Task<TResult> Task => _task;
```

#### 3. 设置内部Task的完成状态是通过调用Task的方法实现的

```cs
public bool TrySetResult(TResult result)
{
    bool rval = _task.TrySetResult(result);
    if (!rval)
    {
        _task.SpinUntilCompleted();
    }

    return rval;
}
```