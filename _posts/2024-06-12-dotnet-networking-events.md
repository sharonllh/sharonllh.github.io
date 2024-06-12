---
layout: post
title: 使用.NET的HttpClient发送请求时，怎么定位延迟？
date: 2024-06-12 +0800
tags: [.NET, HttpClient, 延迟]
categories: [.NET]
---

最近在工作中碰到一个问题，使用.NET的HttpClient发送请求时，发现延迟特别高。这里的延迟指的是从请求发出到收到响应的时间。

导致高延迟的原因可能有多种：
1. 服务器端的处理时间比较长
2. 客户端到服务器端的网络延迟（RTT）比较高
3. 客户端到服务器端的连接比较冷，无法复用连接
4. 客户端需要发送的请求太多了，在请求队列中排队时间比较长

首先排查原因1。服务器端也是我们组管的，所以可以很容易地在响应头中记录服务器端的处理时间。经排查，高延迟不是1导致的。

然后排查原因2。客户端和服务器端可能在不同的地区，RTT受地理位置的影响很大，所以我们根据客户端和服务器端是否在同一个数据中心、同一个地区，进一步计算了延迟。结果发现，即使在同一个数据中心，延迟也比预期的要高（50ms VS. 10ms）。这里10ms是官方文档给出的RTT，不一定是实际情况下的RTT，所以无法排除原因2。

为了进一步定位延迟，我们使用.NET中`EventListener`，记录各种网络事件的时间，包括DNS解析、建立socket连接、建立TLS连接、发送请求和接收响应的时间等。

示例代码如下：

```cs
// HttpEventListener.cs
public class HttpEventListener : EventListener
{
    private static readonly HashSet<string> InterestingEventSources = new HashSet<string>()
    {
        "System.Net.Http",
        "System.Net.NameResolution",
        "System.Net.Sockets",
        "System.Net.Security"
    };

    private readonly AsyncLocal<StringBuilder> logger = new AsyncLocal<StringBuilder>();

    public void SetLogger(StringBuilder logger)
    {
        this.logger.Value = logger;
    }

    protected override void OnEventSourceCreated(EventSource eventSource)
    {
        base.OnEventSourceCreated(eventSource);

        if (InterestingEventSources.Contains(eventSource.Name))
        {
            this.EnableEvents(eventSource, EventLevel.Informational);
        }
    }

    protected override void OnEventWritten(EventWrittenEventArgs eventData)
    {
        base.OnEventWritten(eventData);

        if (this.logger.Value != null)
        {
            this.logger.Value.AppendLine($"[{eventData.TimeStamp}]: {eventData.EventSource.Name}-{eventData.EventName}");
        }
    }
}

// Program.cs
internal class Program
{
    private static readonly HttpClient httpClient = new HttpClient();
    private static readonly HttpEventListener httpEventListener = new HttpEventListener();

    static async Task Main(string[] args)
    {
        await Parallel.ForAsync(0, 5, async (index, _) =>
        {
            var logger = new StringBuilder();
            httpEventListener.SetLogger(logger);

            await Task.Delay(Random.Shared.Next(5000));
            await httpClient.GetAsync("https://www.bing.com/");
            Console.WriteLine($"--------{index}--------\n{logger}");
        });
    }
}
```

根据文档[Guidelines for using HttpClient](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient-guidelines)，推荐使用HttpClient单例。这是因为：
1. 创建HttpClient时，背后会创建一个连接池（connection pool），而析构HttpClient时会释放连接池中的所有连接。如果对每个请求创建一个HttpClient，会导致每次都创建新的连接。
2. 根据TCP协议，连接关闭后不会立马释放TCP端口，所以频繁创建并析构HttpClient会导致端口耗尽（port exhaustion）。

根据文档[Networking events in .NET](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/telemetry/events)，`EventListener`负责监听`EventSource`发出来的事件，其中有两个方法比较重要：
- `OnEventSourceCreated`：每次创建`EventSource`时会被调用一次，可以在这个方法中enable自己感兴趣的`EventSource`。
- `OnEventWritten`：每次`EventSource`发出事件时会被调用，可以在这个方法中记录事件的时间。

由于监听`EventSource`会有额外的性能开销，所以需要注意只enable自己感兴趣的`EventSource`。常见的`EventSource`可以参考这个文档：[Well-known event providers in .NET](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/well-known-event-providers)。

另外，如果创建多个`EventListener`，每个实例都会去监听`EventSource`，造成不必要的开销，所以我们只创建一个单例。

最后，在多线程的情况下，怎么确定某个事件是属于哪个请求的呢？答案是使用`AsyncLocal`。由于使用HttpClient发送请求是异步的，所以需要使用`AsyncLocal`而不是`ThreadLocal`。

上述代码的运行结果如下。可以发现，编号2对应的线程创建了新连接，其他线程则复用了这个连接。

```
--------2--------
[6/12/2024 5:44:03 AM]: System.Net.Http-RequestStart
[6/12/2024 5:44:03 AM]: System.Net.NameResolution-ResolutionStart
[6/12/2024 5:44:03 AM]: System.Net.NameResolution-ResolutionStop
[6/12/2024 5:44:03 AM]: System.Net.Sockets-ConnectStart
[6/12/2024 5:44:03 AM]: System.Net.Sockets-ConnectStop
[6/12/2024 5:44:03 AM]: System.Net.Security-HandshakeStart
[6/12/2024 5:44:03 AM]: System.Net.Security-HandshakeStop
[6/12/2024 5:44:03 AM]: System.Net.Http-ConnectionEstablished
[6/12/2024 5:44:03 AM]: System.Net.Http-RequestLeftQueue
[6/12/2024 5:44:03 AM]: System.Net.Http-RequestHeadersStart
[6/12/2024 5:44:03 AM]: System.Net.Http-RequestHeadersStop
[6/12/2024 5:44:03 AM]: System.Net.Http-ResponseHeadersStart
[6/12/2024 5:44:03 AM]: System.Net.Http-ResponseHeadersStop
[6/12/2024 5:44:03 AM]: System.Net.Http-ResponseContentStart
[6/12/2024 5:44:03 AM]: System.Net.Http-ResponseContentStop
[6/12/2024 5:44:03 AM]: System.Net.Http-RequestStop

--------0--------
[6/12/2024 5:44:04 AM]: System.Net.Http-RequestStart
[6/12/2024 5:44:04 AM]: System.Net.Http-RequestHeadersStart
[6/12/2024 5:44:04 AM]: System.Net.Http-RequestHeadersStop
[6/12/2024 5:44:04 AM]: System.Net.Http-ResponseHeadersStart
[6/12/2024 5:44:04 AM]: System.Net.Http-ResponseHeadersStop
[6/12/2024 5:44:04 AM]: System.Net.Http-ResponseContentStart
[6/12/2024 5:44:04 AM]: System.Net.Http-ResponseContentStop
[6/12/2024 5:44:04 AM]: System.Net.Http-RequestStop

--------4--------
[6/12/2024 5:44:04 AM]: System.Net.Http-RequestStart
[6/12/2024 5:44:04 AM]: System.Net.Http-RequestHeadersStart
[6/12/2024 5:44:04 AM]: System.Net.Http-RequestHeadersStop
[6/12/2024 5:44:04 AM]: System.Net.Http-ResponseHeadersStart
[6/12/2024 5:44:04 AM]: System.Net.Http-ResponseHeadersStop
[6/12/2024 5:44:04 AM]: System.Net.Http-ResponseContentStart
[6/12/2024 5:44:04 AM]: System.Net.Http-ResponseContentStop
[6/12/2024 5:44:04 AM]: System.Net.Http-RequestStop

--------3--------
[6/12/2024 5:44:05 AM]: System.Net.Http-RequestStart
[6/12/2024 5:44:05 AM]: System.Net.Http-RequestHeadersStart
[6/12/2024 5:44:05 AM]: System.Net.Http-RequestHeadersStop
[6/12/2024 5:44:05 AM]: System.Net.Http-ResponseHeadersStart
[6/12/2024 5:44:05 AM]: System.Net.Http-ResponseHeadersStop
[6/12/2024 5:44:05 AM]: System.Net.Http-ResponseContentStart
[6/12/2024 5:44:05 AM]: System.Net.Http-ResponseContentStop
[6/12/2024 5:44:05 AM]: System.Net.Http-RequestStop

--------1--------
[6/12/2024 5:44:05 AM]: System.Net.Http-RequestStart
[6/12/2024 5:44:05 AM]: System.Net.Http-RequestHeadersStart
[6/12/2024 5:44:05 AM]: System.Net.Http-RequestHeadersStop
[6/12/2024 5:44:06 AM]: System.Net.Http-ResponseHeadersStart
[6/12/2024 5:44:06 AM]: System.Net.Http-ResponseHeadersStop
[6/12/2024 5:44:06 AM]: System.Net.Http-ResponseContentStart
[6/12/2024 5:44:06 AM]: System.Net.Http-ResponseContentStop
[6/12/2024 5:44:06 AM]: System.Net.Http-RequestStop
```