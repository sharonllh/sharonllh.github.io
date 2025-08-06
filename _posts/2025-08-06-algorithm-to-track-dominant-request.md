---
layout: post
title: 一种实时统计超高频请求的简单算法
date: 2025-08-06 +0800
tags: [实时统计, 最高频]
categories: [系统设计]
---

最近看同事的代码，发现他用了一种非常简单的实时统计超高频请求的算法，在此记录一下。

具体的需求是，在一个Web API中，实时统计超高频请求——即数量远超于其他所有请求的请求。当超高频请求的数量超过一定阈值时，触发快速返回响应的机制。

对于这个需求，我的第一个想法就是用一个`ConcurrentDictionary`来记录请求，`key`是请求的pattern，`value`是请求的数量。实现的时候，需要注意限制该字典的大小，避免占用的内存过大。

这个方法已经很简单了，但是同事用了一种更加简单的算法，具体如下：

1. 初始化两个变量，用于记录超高频请求的信息：
    ```cs
    // 超高频请求的pattern
    string dominantRequest = string.Empty;

    // 超高频请求的数量
    int dominantCount = 0;
    ```

2. 对于每一个请求，更新超高频请求的信息：
    ```cs
    if (dominantCount == 0)
    {
        // 把当前请求临时标记为超高频请求
        dominantRequest = request;
        dominantCount = 1;
    }
    else if (dominantRequest == request)
    {
        dominantCount++;
    }
    else
    {
        dominantCount--;
    }
    ```

如果存在一个数量远超于其他所有请求的超高频请求，用上面这种算法一定会达到一个`dominantCount`超级大的状态，因此后面再判断`dominantCount`的大小，并进行相应的处理就可以了。