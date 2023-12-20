---
layout: post
title: .NET的Regex类是线程安全的吗？
date: 2023-10-16 +0800
tags: [.NET, Regex, 线程安全]
categories: [.NET]
---

在.NET中，[Regex类](https://learn.microsoft.com/en-us/dotnet/api/system.text.regularexpressions.regex)表示正则表达式。这个类是线程安全的吗？最近碰到了一个案例，对这个问题有了一些理解，在此记录一下。

## 案例描述

我们有一个REST API，托管了`/{CompanyId}/Items/{ItemId}`和`/{CompanyId}/Items(ItemUniqueName='{ItemUniqueName}')`等多个endpoints，其中`{}`包含的是参数，不同的request会有不同的值。

对于每个request，我们想提取item identity，方法如下：

1. 每个endpoint对应固定template的item identity，例如`/{CompanyId}/Items/{ItemId}`对应的template是`IID:{ItemId}@{CompanyId}`，`/{CompanyId}/Items(ItemUniqueName='{ItemUniqueName}')`对应的template是`IUN:{ItemUniqueName}@{CompanyId}`。第一步需要确认template。
2. 第二步将template中的参数（如`{CompanyId}`和`{ItemId}`）替换成具体的值，这样就能得到item identity。

举个例子，对于request `/403545c9-707c-4df4-965c-073f3981c78e/Items/c0230fe9-9899-4f78-a8e4-585e2d9521fc`，相应的template是`IID:{ItemId}@{CompanyId}`，替换参数后得到item identity为`IID:c0230fe9-9899-4f78-a8e4-585e2d9521fc@403545c9-707c-4df4-965c-073f3981c78e`。

提取item identity的代码如下：

```cs
// Parameter pattern. For example, "{ItemId}" and "{CompanyId}" in "IID:{ItemId}@{CompanyId}".
private static readonly Regex ParameterPattern
    = new Regex("{([a-z_A-Z0-9]*)}", RegexOptions.Compiled);

// Cache for regex match results.
private static readonly ConcurrentDictionary<string, MatchCollection> TemplateToRegexMatchResultMapping
    = new ConcurrentDictionary<string, MatchCollection>(StringComparer.OrdinalIgnoreCase);

public static string ParseItemIdentity(HttpContext httpContext, string template)
{
    // Firstly parse the parameters of the template
    var allMatches = TemplateToRegexMatchResultMapping.GetOrAdd(
        template,
        ParameterPattern.Matches(template));
    
    // Then replace the parameters with the values in httpContext
    StringBuilder stringBuilder = new StringBuilder(
        GetStringBeforeFirstParameter(template, allMatches[0]));
    for (int i = 0; i < allMatches.Count; i++)
    {
        stringBuilder.Append(GetParameterValue(allMatches[i], httpContext));
        stringBuilder.Append(GetStringBetweenParameters(template, allMatches[i], i));
    }

    // Finally get the item identity
    return stringBuilder.ToString();
}
```

在`ParseItemIdentity`方法中，我们首先正则匹配template中的所有参数。由于template是固定而有限的，我们做了一个优化，把每一个template和对应的匹配结果缓存起来，这样就可以节省掉正则匹配的时间。为了保证线程安全，我们用了`ConcurrentDictionary`作为缓存。

到这里似乎一切都很完美。然而，到了线上，却出现了大量错误的item identity，本来应该是`IID:{ItemId}@{CompanyId}`，结果却是`IID:{ItemId}@{CompanyId}{ItemId}@{CompanyId}`。

## 问题分析

在错误的item identity中`{ItemId}@{CompanyId}`被重复了两遍，所以我们的第一想法是，会不会`ParseItemIdentity`中替换参数的代码有问题？检查之后发现没有问题。

进一步分析发现，错误的item identity只出现在部分机器上，而且在出问题的机器上，并不是所有的item identity都有问题。某个template对应的item identity要么都正确，要么都错误。一旦开始出现错误的item identity，就会一直持续，直到服务被重启。

结合这些发现，我们怀疑会不会是多线程导致`TemplateToRegexMatchResultMapping`被corrupt了？我们用到了`ConcurrentDictionary.GetOrAdd`方法，根据[官方文档](https://learn.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentdictionary-2.getoradd?view=net-7.0)的描述，这个方法是线程安全的，但是valueFactory的调用是不加锁的，可能会被多个线程同时调用。

```
For modifications and write operations to the dictionary, ConcurrentDictionary<TKey,TValue> uses fine-grained locking to ensure thread safety. (Read operations on the dictionary are performed in a lock-free manner.) However, the valueFactory delegate is called outside the locks to avIID the problems that can arise from executing unknown code under a lock. Therefore, GetOrAdd is not atomic with regards to all other operations on the ConcurrentDictionary<TKey,TValue> class.

Since a key/value can be inserted by another thread while valueFactory is generating a value, you cannot trust that just because valueFactory executed, its produced value will be inserted into the dictionary and returned. If you call GetOrAdd simultaneously on different threads, valueFactory may be called multiple times, but only one key/value pair will be added to the dictionary.
```

也就是说，`TemplateToRegexMatchResultMapping`本身并没有corrupt，但是`ParameterPattern.Matches(template)`这个操作是不加锁的，可能不是线程安全的。那么`Regex`类是不是线程安全的呢？

根据[Thread Safety in Regular Expressions](https://learn.microsoft.com/en-us/dotnet/standard/base-types/thread-safety-in-regular-expressions)中的描述，`Regex`类本身是线程安全的，但是正则匹配的结果`Match`和`MatchCollection`不是。我们的代码里缓存了`MatchCollection`，会导致多个线程同时使用一个`MatchCollection`，从而产生错误。

```
The Regex class itself is thread safe and immutable (read-only). That is, Regex objects can be created on any thread and shared between threads; matching methods can be called from any thread and never alter any global state.

However, result objects (Match and MatchCollection) returned by Regex should be used on a single thread. Although many of these objects are logically immutable, their implementations could delay computation of some results to improve performance, and as a result, callers must serialize access to them.

If there is a need to share Regex result objects on multiple threads, these objects can be converted to thread-safe instances by calling their synchronized methods. With the exception of enumerators, all regular expression classes are thread safe or can be converted into thread-safe objects by a synchronized method.

Enumerators are the only exception. An application must serialize calls to collection enumerators. The rule is that if a collection can be enumerated on more than one thread simultaneously, you should synchronize enumerator methods on the root object of the collection traversed by the enumerator.
```

为了验证这个猜想，我们在出问题的机器上抓取了一个dump，可以看到`MatchCollection`中确实包含了错误的匹配结果：应该匹配到2个参数，却匹配到了4个参数。
![Dump](/assets/img/RegexThreadSafety_Dump.png)

## 源码解析

从源码可以很清楚地看出`MatchCollection`不是线程安全的。

1. 调用`Regex.Matches`方法时，不会立即执行正则匹配，仅仅只是生成一个`MatchCollection`。

    ```cs
    public MatchCollection Matches(string input)
    {
        if (input is null)
        {
            ThrowHelper.ThrowArgumentNullException(ExceptionArgument.input);
        }
    
        return new MatchCollection(this, input, RightToLeft ? input.Length : 0);
    }
    ```

2. 枚举`MatchCollection`时，才会真正执行正则匹配，并且每次枚举只会找一个匹配，而不是将所有匹配全部找出来。`MatchCollection`内部有一个`_matches`列表用来缓存匹配结果。

    ```cs
    public class MatchCollection : IList<Match>, IReadOnlyList<Match>, IList
    {
        private readonly Regex _regex;
        private readonly List<Match> _matches;
        private readonly string _input;
        private int _startat;
        private int _prevlen;
        private bool _done;

        private Match? GetMatch(int i)
        {
            Debug.Assert(i >= 0, "i cannot be negative.");
    
            if (_matches.Count > i)
            {
                return _matches[i];
            }
    
            if (_done)
            {
                return null;
            }
    
            Match match;
            do
            {
                match = _regex.RunSingleMatch(RegexRunnerMode.FullMatchRequired, _prevlen, _input, 0, _input.Length, _startat)!;
                if (!match.Success)
                {
                    _done = true;
                    return null;
                }
    
                _matches.Add(match);
                _prevlen = match.Length;
                _startat = match._textpos;
            } while (_matches.Count <= i);
    
            return match;
        }

        private sealed class Enumerator : IEnumerator<Match>
        {
            private readonly MatchCollection _collection;
            private int _index;
    
            public bool MoveNext()
            {
                if (_index == -2)
                {
                    return false;
                }
    
                _index++;
                Match? match = _collection.GetMatch(_index);
    
                if (match is null)
                {
                    _index = -2;
                    return false;
                }
    
                return true;
            }
    
            public Match Current
            {
                get
                {
                    if (_index < 0)
                    {
                        throw new InvalidOperationException(SR.EnumNotStarted);
                    }
    
                    return _collection.GetMatch(_index)!;
                }
            }
        }
    }
    ```

## 解决方法

解决方法包括两个方面：
1. 处理`MatchCollection`：提前枚举`MatchCollection`，触发正则匹配的执行。
2. 处理`Match`：对于每个`Match`，调用`Synchronized`方法让它变得线程安全。

代码如下：

```cs
// Cache for regex match results.
private static readonly ConcurrentDictionary<string, IList<Match>> TemplateToRegexMatchResultMapping = new ConcurrentDictionary<string, IList<Match>>(StringComparer.OrdinalIgnoreCase);

public string ParseItemIdentity(HttpContext httpContext, string template)
{
    var allMatches = TemplateToRegexMatchResultMapping.GetOrAdd(
        template,
        ParameterPattern.Matches(template).Select(match => Match.Synchronized(match)).ToArray());

    // ... ...
}

```