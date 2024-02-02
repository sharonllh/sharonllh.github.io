---
layout: post
title: LINQ中的Aggregate方法怎么用？
date: 2024-02-02 +0800
tags: [.NET, LINQ, Aggregate]
categories: [.NET]
---

最近在写代码的时候，GitHub Copilot提示的代码中使用了LINQ的`Aggregate`方法，以前没用过，所以研究了一下。

## Aggregate方法的用处

我们从一个例子来看`Aggregate`方法的用处。假设我们有一个字典，记录了班级里所有同学的数学成绩，然后我们想找到获得最高分的同学是哪位，代码如下：

```cs
IDictionary<string, int> mathScores = new Dictionary<string, int>();
mathScores.Add("Lily", 97);
mathScores.Add("David", 89);
mathScores.Add("Lei", 100);
mathScores.Add("Lucy", 78);

KeyValuePair<string, int> highestScore = mathScores.Aggregate(
    (highest, next) => highest.Value > next.Value ? highest : next);
Console.WriteLine($"The highest score is {highestScore.Value} held by {highestScore.Key}.");
```

这段代码用了`Aggregate`方法的一个重载，它是这样工作的：首先把`highest`的初始值设为`mathScores`的第一个元素，然后遍历`mathScores`中的剩余元素（不包括第一个元素），依次和当前的`highest`做比较，不断更新`highest`。遍历结束后，返回当前的`highest`。

这个重载有两个限制：
1. 如果`mathScores`为空，会抛`System.InvalidOperationException`。
2. 中间变量`highest`和字典元素必须是同一个类型的。

假如我们想处理`mathScores`为空的情况，可以使用另一个重载来手动设置`highest`的初始值，从而避免抛异常，代码如下：

```cs
KeyValuePair<string, int> highestScore = mathScores.Aggregate(
    new KeyValuePair<string, int>(string.Empty, -1),
    (highest, next) => highest.Value > next.Value ? highest : next);
```

这个重载方法会把`highest`的初始值设置成第一个参数，然后按照第二个参数依次遍历`mathScores`。如果`mathScores`为空，就会返回初始值，而不会抛异常。

另外，使用这个重载方法的时候，中间变量`highest`可以是任意类型，不仅限于元素类型。举个例子，假如我们想同时找到最高分和最低分，该怎么做呢？

```cs
IDictionary<string, int> mathScores = new Dictionary<string, int>();
mathScores.Add("Lily", 97);
mathScores.Add("David", 89);
mathScores.Add("Lei", 100);
mathScores.Add("Lucy", 78);

var highestAndLowestScores = mathScores.Aggregate(
    (new KeyValuePair<string, int>(string.Empty, -1), new KeyValuePair<string, int>(string.Empty, -1)),
    (highestAndLowest, next) =>
    {
        var highest = (highestAndLowest.Item1.Value == -1 || highestAndLowest.Item1.Value < next.Value)
            ? next
            : highestAndLowest.Item1;

        var lowest = (highestAndLowest.Item2.Value == -1 || highestAndLowest.Item2.Value > next.Value)
            ? next
            : highestAndLowest.Item2;

        return (highest, lowest);
    });

Console.WriteLine(
    $"The highest score is {highestAndLowestScores.Item1.Value} held by {highestAndLowestScores.Item1.Key}, " +
    $"while the lowest score is {highestAndLowestScores.Item2.Value} held by {highestAndLowestScores.Item2.Key}.");
```

最后，如果我们只需要返回获得最高分和最低分的同学姓名，而不需要具体分数的话，可以使用第三个重载方法，设置返回类型。代码如下：

```cs
var highestAndLowestScorePersons = mathScores.Aggregate(
    (new KeyValuePair<string, int>(string.Empty, -1), new KeyValuePair<string, int>(string.Empty, -1)),
    (highestAndLowest, next) =>
    {
        var highest = (highestAndLowest.Item1.Value == -1 || highestAndLowest.Item1.Value < next.Value)
            ? next
            : highestAndLowest.Item1;

        var lowest = (highestAndLowest.Item2.Value == -1 || highestAndLowest.Item2.Value > next.Value)
            ? next
            : highestAndLowest.Item2;

        return (highest, lowest);
    },
    highestAndLowest => (highestAndLowest.Item1.Key, highestAndLowest.Item2.Key));
```

## 源码解析

`Aggregate`方法的源代码没什么复杂的，以参数最多的重载方法为例，它只是提供了一个遍历集合并比较元素的语法糖。

```cs
public static TResult Aggregate<TSource, TAccumulate, TResult>(
    this IEnumerable<TSource> source,
    TAccumulate seed,
    Func<TAccumulate, TSource, TAccumulate> func,
    Func<TAccumulate, TResult> resultSelector)
{
    if (source == null)
    {
        ThrowHelper.ThrowArgumentNullException(ExceptionArgument.source);
    }
 
    if (func == null)
    {
        ThrowHelper.ThrowArgumentNullException(ExceptionArgument.func);
    }
 
    if (resultSelector == null)
    {
        ThrowHelper.ThrowArgumentNullException(ExceptionArgument.resultSelector);
    }
 
    TAccumulate result = seed;
    foreach (TSource element in source)
    {
        result = func(result, element);
    }
 
    return resultSelector(result);
}
```

比较值得借鉴的是参数的设计，尤其是参数`Func<TAccumulate, TSource, TAccumulate> func`的设计，有点数学归纳法的意思。

当我们需要遍历集合并比较元素的时候，如果逻辑比较简单，用这个方法会让代码很简洁，但是逻辑复杂的话就没必要用它了。另外，使用这个方法时，给变量起一个有意义的名字，可以让逻辑更清晰。