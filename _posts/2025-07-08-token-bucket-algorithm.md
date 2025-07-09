---
layout: post
title: Token Bucket算法
date: 2025-07-08 +0800
tags: [限流算法]
categories: [系统设计]
---

对于一个Web API，限流（throttling）是一个很重要的组件，主要有两个作用：
1. 限制总体流量，从而保护Web API本身以及底层的组件。
2. 限制每个用户的流量，从而避免一个用户的大量访问影响到其他用户，保证公平性。

Token Bucket算法是一个非常经典的限流算法。

## Token Bucket算法介绍

如图所示，Token Bucket算法会维护一个装有tokens的桶，桶的容量是有限的，并且每隔一段时间会补充tokens。

当一个请求到达时，首先尝试从桶里获取一个或多个tokens，如果成功，就继续执行，否则就拒绝掉这个请求。通过这种方式，就可以限制请求的数量。

![TokenBucket](/assets/img/TokenBucket.webp)

Token Bucket算法有两个重要的参数：
- Capacity：桶的容量，即tokens的最大数量
- Refill Rate：补充tokens的速度

在实际使用时，Refill Rate对应于请求的平均数量，而Capacity对应于瞬时可以处理的最大请求量。Capacity通常比Refill Rate大一些。

另外，在实际使用时，可以为不同的场景设置不同的Token Bucket，从而满足不同场景的限流需求。

## Token Bucket算法实现

Token Bucket算法的C#代码实现如下。

### TokenBucketOptions类

首先，定义一个`TokenBucketOptions`类，负责定义算法的关键参数，即Capacity和Refill Rate。

```cs
public class TokenBucketOptions
{
    public int Capacity { get; }

    public int RefillRatePerInterval { get; }

    public TimeSpan RefillInterval { get; }

    public TokenBucketOptions(int capacity, int refillRatePerInterval, TimeSpan refillInterval)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(capacity);
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(refillRatePerInterval);
        if (refillInterval == TimeSpan.Zero)
        {
            throw new ArgumentException("Invalid refill interval!", nameof(refillInterval));
        }

        this.Capacity = capacity;
        this.RefillRatePerInterval = refillRatePerInterval;
        this.RefillInterval = refillInterval;
    }

    public double RefillRatePerMs
    {
        get
        {
            return this.RefillRatePerInterval / this.RefillInterval.TotalMilliseconds;
        }
    }
}
```

### TokenBucket类

然后，定义`ITokenBucket`接口和`TokenBucket`类，负责实现算法。

```cs
public interface ITokenBucket
{
    bool TryConsume(int requestedTokens);
}

public class TokenBucket : ITokenBucket, IDisposable
{
    private readonly int capacity;
    private readonly int refillRatePerInterval;
    private readonly TimeSpan refillInterval;
    private readonly Timer refiller;
    private readonly object lockObj = new object();

    private int tokens;
    private DateTime whenLastRefilled;

    public TokenBucket(TokenBucketOptions options)
    {
        ArgumentNullException.ThrowIfNull(nameof(options));

        this.capacity = options.Capacity;
        this.refillRatePerInterval = options.RefillRatePerInterval;
        this.refillInterval = options.RefillInterval;

        this.tokens = this.capacity;
        this.whenLastRefilled = DateTime.UtcNow;
        this.refiller = new Timer(_ => this.Refill(), null, dueTime: this.refillInterval, period: this.refillInterval);
    }

    public bool TryConsume(int requestedTokens)
    {
        lock (this.lockObj)
        {
            if (this.tokens >= requestedTokens)
            {
                this.tokens -= requestedTokens;
                return true;
            }

            return false;
        }
    }

    public void Refill()
    {
        lock (this.lockObj)
        {
            if (this.tokens < this.capacity)
            {
                int refillTokens = (int)(this.refillRatePerInterval * ((DateTime.UtcNow - this.whenLastRefilled) / this.refillInterval));
                this.tokens += refillTokens;
                if (this.tokens > this.capacity)
                {
                    this.tokens = this.capacity;
                }

                this.whenLastRefilled = DateTime.UtcNow;
            }
        }
    }

    public override string ToString()
    {
        return $"Tokens: {this.tokens}, WhenLastRefilled: {this.whenLastRefilled}";
    }

    public void Dispose()
    {
        this.refiller.Dispose();
    }
}
```

- 在`TokenBucket`类的内部，维护一个当前的tokens，以及一个background的timer，负责定期补充tokens。
- `TokenBucket`类对外暴露一个`TryConsume`接口，让调用者尝试获取特定数量的tokens。

在上面的实现中，我们定义了一个timer去自动补充tokens，也可以考虑对外暴露`Refill`接口，让调用者在获取tokens前先去主动补充tokens。

## Token Bucket算法应用

### 场景介绍

假设我们有一个Web API，想利用Token Bucket算法实现以下限流：
1. 对所有endpoints有一个总的限流。
2. 对每个endpoint有一个单独的限流。
3. 对每个endpoint，针对每个用户还有一个单独的限流，从而防止一个用户的过量请求影响到其他用户。

为了实现上面的需求，我们需要设置3类（而不是3个）Token Bucket，并按照顺序依次进行限流（3 -> 2 -> 1）。

下面是C#代码实现。

### ThrottlePolicy类

首先，定义两个类来表示限流策略，其中：
- `GlobalThrottlePolicy`表示Global的限流策略，即所有请求共享的策略。
- `PartitionedThrottlePolicy`表示分区的限流策略，这里的分区可以是根据endpoint的分区、根据用户的分区。

```cs
public abstract class ThrottlePolicyBase
{
    public string Name { get; init; }

    public TokenBucketOptions TokenBucketOptions { get; init; }

    public int MinRetryAfterMs { get; init; }

    public int MaxRetryAfterMs { get; init; }
}

public class GlobalThrottlePolicy : ThrottlePolicyBase
{
}

public class PartitionedThrottlePolicy : ThrottlePolicyBase
{
    public Func<HttpContext, string> PartitionKeyGetter { get; init; }
}
```

### Throttler类

然后，定义限流器的类。这些类会根据限流策略创建Token Bucket，然后根据是否能从Token Bucket中获取tokens来判断一个请求是否需要限流。

#### IThrottler接口

限流器的接口如下，输入参数是`HttpContext`，输出包括是否执行限流、触发限流的原因和需要采取的措施等。

```cs
public class ThrottleResult
{
    public string ThrottlePolicyName { get; init; }

    public int RetryAfterMs { get; init; }
}

public interface IThrottler
{
    bool TryThrottle(HttpContext context, out ThrottleResult result);
}
```

#### GlobalThrottler类

Global的限流器如下，由于所有的请求共享一个Token Bucket，可以直接在构造器中创建这个Bucket，然后在`TryThrottle`中使用即可。

```cs
public class GlobalThrottler : IThrottler
{
    private readonly GlobalThrottlePolicy policy;
    private readonly ITokenBucket tokenBucket;

    public GlobalThrottler(GlobalThrottlePolicy policy)
    {
        ArgumentNullException.ThrowIfNull(policy);
        ArgumentException.ThrowIfNullOrWhiteSpace(policy.Name);
        ArgumentNullException.ThrowIfNull(policy.TokenBucketOptions);

        this.policy = policy;
        this.tokenBucket = new TokenBucket(policy.TokenBucketOptions);
    }

    public bool TryThrottle(HttpContext context, out ThrottleResult result)
    {
        Console.WriteLine($"[TokenBucket] {tokenBucket}");
        bool shouldThrottle = !this.tokenBucket.TryConsume(requestedTokens: 1);

        if (shouldThrottle)
        {
            result = new ThrottleResult()
            {
                ThrottlePolicyName = this.policy.Name,
                RetryAfterMs = RetryAfterCalculator.CalculateRetryAfterMs(this.policy.TokenBucketOptions.RefillRatePerMs, this.policy.MinRetryAfterMs, this.policy.MaxRetryAfterMs)
            };

            return true;
        }

        result = null;
        return false;
    }
}
```

#### PartitionedThrottler类

分区的限流器实现如下，由于每个分区都需要有自己的Token Bucket，所以我们无法在构造器中事先创建所有的Bucket，而需要在`TryConsume`中按需创建。

```cs
public class PartitionedThrottler : IThrottler
{
    private readonly PartitionedThrottlePolicy policy;
    private readonly ConcurrentDictionary<string, TokenBucket> tokenBuckets;

    public PartitionedThrottler(PartitionedThrottlePolicy policy)
    {
        ArgumentNullException.ThrowIfNull(policy);
        ArgumentException.ThrowIfNullOrWhiteSpace(policy.Name);
        ArgumentNullException.ThrowIfNull(policy.PartitionKeyGetter);
        ArgumentNullException.ThrowIfNull(policy.TokenBucketOptions);

        this.policy = policy;
        this.tokenBuckets = new ConcurrentDictionary<string, TokenBucket>(StringComparer.OrdinalIgnoreCase);
    }

    public bool TryThrottle(HttpContext context, out ThrottleResult result)
    {
        string partitionKey = this.policy.PartitionKeyGetter(context);
        TokenBucket tokenBucket = this.tokenBuckets.GetOrAdd(partitionKey, new TokenBucket(this.policy.TokenBucketOptions));
        Console.WriteLine($"[TokenBucket] {tokenBucket}, PolicyName: {this.policy.Name}, PartitionKey: {partitionKey}");
        bool shouldThrottle = !tokenBucket.TryConsume(requestedTokens: 1);

        if (shouldThrottle)
        {
            result = new ThrottleResult()
            {
                ThrottlePolicyName = this.policy.Name,
                RetryAfterMs = RetryAfterCalculator.CalculateRetryAfterMs(this.policy.TokenBucketOptions.RefillRatePerMs, this.policy.MinRetryAfterMs, this.policy.MaxRetryAfterMs)
            };

            return true;
        }

        result = null;
        return false;
    }
}
```

### RetryAfterCalculator类

当一个请求被限流后，通常我们希望客户端等待一段时间后再重试，而不是立马重试。对此，我们可以在返回的响应中添加一个`X-RetryAfter` header，让客户端根据这个值等待。

RetryAfter的值跟Token Bucket的Refill Rate成反比，Refill得越快，则需要等待得的时间越短。另外，RetryAfter的值应该是随机的，避免同一时间被限流的请求又在同一时间重试。

RetryAfter的计算逻辑如下。

```cs
public class RetryAfterCalculator
{
    public static int CalculateRetryAfterMs(double tokenBucketRefillRatePerMs, int minRetryAfterMs, int maxRetryAfterMs)
    {
        return (int)(1 / tokenBucketRefillRatePerMs) + Random.Shared.Next(minRetryAfterMs, maxRetryAfterMs);
    }
}
```

### ThrottleMiddleware

最后，定义一个middleware，用来执行真正的限流操作。我们根据需求创建了3个限流器，并让每个请求都经过这3个限流器进行判断。

```cs
public class ThrottleMiddleware
{
    private readonly RequestDelegate next;
    private readonly IThrottler globalThrottler;
    private readonly IThrottler perApiThrottler;
    private readonly IThrottler perUserPerApiThrottler;

    public ThrottleMiddleware(RequestDelegate next)
    {
        ArgumentNullException.ThrowIfNull(next);
        this.next = next;

        this.globalThrottler = new GlobalThrottler(new GlobalThrottlePolicy()
        {
            Name = "GlobalThrottle",
            TokenBucketOptions = new TokenBucketOptions(7000, 6000, TimeSpan.FromMinutes(1)),
            MinRetryAfterMs = 10,
            MaxRetryAfterMs = 20
        });

        this.perApiThrottler = new PartitionedThrottler(new PartitionedThrottlePolicy()
        {
            Name = "PerApiThrottle",
            PartitionKeyGetter = context => context.GetEndpoint()?.DisplayName ?? "Unknown",
            TokenBucketOptions = new TokenBucketOptions(4000, 3000, TimeSpan.FromMinutes(1)),
            MinRetryAfterMs = 5,
            MaxRetryAfterMs = 10
        });

        this.perUserPerApiThrottler = new PartitionedThrottler(new PartitionedThrottlePolicy()
        {
            Name = "PerUserPerApiThrottle",
            PartitionKeyGetter = context => context.Request.Headers["User-Agent"] + "|" + context.GetEndpoint()?.DisplayName ?? "Unknown",
            TokenBucketOptions = new TokenBucketOptions(800, 600, TimeSpan.FromMinutes(1)),
            MinRetryAfterMs = 1,
            MaxRetryAfterMs = 5
        });
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (this.perUserPerApiThrottler.TryThrottle(context, out ThrottleResult result)
            || perApiThrottler.TryThrottle(context, out result)
            || globalThrottler.TryThrottle(context, out result))
        {
            context.Response.StatusCode = 429;
            context.Response.Headers["X-RetryAfterMs"] = result.RetryAfterMs.ToString();
            await context.Response.WriteAsync($"The request is throttled by '{result.ThrottlePolicyName}' policy, please retry after {result.RetryAfterMs} ms.");
            return;
        }

        await this.next(context);
    }
}
```

## 其他考虑

1. 本文的Token Bucket是利用timer来自动填充tokens的，也可以考虑让调用者在每次尝试获取tokens前先去手动填充。
2. 本文会直接拒绝掉被限流的请求，也可以考虑把它们先放到一个队列里，等tokens填充后再去队列里取出进行处理。
3. 尽管提供了X-RetryAfter让客户端过一会儿再重试，但这并不是强制的，客户端可能无视这个值直接无脑重试，从而导致我们的服务更加繁忙。对此，可以考虑延迟一段时间再返回被限流的请求。