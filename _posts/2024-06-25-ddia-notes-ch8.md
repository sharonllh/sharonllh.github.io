---
layout: post
title: DDIA阅读笔记(8)：分布式数据系统的麻烦
date: 2024-06-25 +0800
tags: [系统设计, DDIA]
categories: [系统设计]
---

## 不可靠的网络

互联网和以太网中的大多数内部网络是异步分组网络，一个节点向另一个节点发送消息时，网络不能保证是否到达、什么时候到达。因此，发送请求时，需要设置超时（timeout），在一段时候后放弃等待。

网络故障是很常见的，软件需要有能力应对网络故障，并从中恢复。为了保证软件的这方面能力，可以采用Chaos Monkey的想法，有意识地触发网络问题并测试系统响应。

### 检测节点故障

许多系统需要自动检测节点故障，例如：
- 负载均衡器需要停止向已经死亡的节点转发请求。
- 在单主复制的数据库中，如果主库失效，需要将从库之一升级成主库。

在以下情况中，可以明确判断节点发生故障了：
- 如果可以连接到运行节点的机器，但没有进程在监听目标端口，那么操作系统会发送FIN或RST来关闭并重用TCP连接。
- 如果节点进程崩溃，但操作系统仍然在运行，那么可以通过脚本通知其他节点关于该崩溃的信息，从而让另一个节点快速接管。
- 在数据中心网络交换机的管理界面，可以检测硬件级别的链路故障，如远程机器是否关闭电源等。
- 如果路由器确认要连接的IP地址不可用，可能会返回ICMP目标不可达数据包。

在其他情况下，可能没有这么明确的信号。这时候检测故障可以采取这样的办法：设置一个超时和重试次数，如果连续几次都无法在超时时间内收到响应，就认为节点已经死亡。

### 设置超时

首先，超时不能设置得太长。如果超时太长了，那么很长时间才能知道一个节点故障了，这段时间内用户必须等待。

其次，超时不能设置得太短。如果超时太短了，固然可以更快检测到故障，但是也有可能会误判。
- 如果一个节点没有死亡，还在执行发邮件等操作，却被误判为故障，由另一个节点接管。那么，发邮件这个操作会被执行两次。
- 如果一个节点没有死亡，只是处于高负载的状态，却被误判为故障，由另一个节点接管。那么，转移过来的负载可能会导致其他节点出现级联失效。

如果网络的最大延迟为`d`，服务器的最大处理时间为`r`，那么可以设置超时为`2d + r`。可惜在现实中，异步网络具有无限延迟，服务器也不保证在一定时间内处理请求。

可以通过实验方法选择超时：在较长的一段时间内，在多台机器上测量响应时间的分布，基于这个数据选择超时。另外，超时可以是不固定的，而是根据响应时间的变化自动调整的。

## 不可靠的时钟

每台机器的时钟是由一个硬件设备决定的：石英晶体振荡器。由于这些设备不是完全准确的，所以每台机器都有自己的时间。

可以采用网络时间协议（NTP）来同步时钟。NTP服务器从精确的时间源（如GPS接收机）获取时间，然后每台机器可以根据一组服务器报告的时间来调整时钟。

### 单调钟和日历时钟

现代计算机至少有两种时钟：
- 日历时钟（time-of-day clock）
- 单调钟（monotonic clock）

#### 日历时钟

日历时钟是根据某个日历（通常是1970年1月1日午夜）返回的当前日期和时间。

日历时钟与NTP同步，从而使不同机器上的时间相同。如果本地时钟和NTP服务器相差较远，可能会被强制重置，有时候看上去好像时间往回跳了一样。

由于时间跳跃和忽略闰秒的原因，日历时钟不能用来测量elapsed time。

#### 单调钟

单调钟可能是计算机启动以来的纳秒数，或者类似的任意值。

单调钟的绝对值是没有意义的，可以用两个绝对值之间的差异来衡量持续时间。

NTP不能让单调钟跳转，只能调整单调钟向前走的频率。

单调钟没有时间跳跃，分辨率也很高（微秒级），所以可以用来测量elapsed time。

### 时钟同步和准确性

单调钟不需要同步，日历时钟需要NTP来同步。然而，即使有NTP同步，日历时钟也不一定是准确的，原因可能有：
- 时钟漂移：计算机中的石英钟不准确，运行速度偏快或偏慢。
- 如果计算机的时钟和NTP服务器的时钟相差较大，可能会拒绝同步，或者被强制重置。
- 某个节点可能会被NTP服务器的防火墙意外阻塞。
- 网络延迟会限制NTP同步的准确性。
- 某些NTP服务器可能是错误的。NTP客户端可以查询多个服务器并忽略异常值，从而避免设置错误的值。
- 系统可能没有考虑闰秒。
- 用户可能会故意把时钟调整为错误的值。

通过GPS接收机、精确时间协议（PTP）、仔细的部署和监测，可以实现很好的时钟精度。

### 依赖时钟同步

不正确的时钟不会造成系统崩溃，所以容易被忽略。

如果使用需要同步时钟的软件，必须监控所有机器之间的时钟偏移。如果一个节点的时钟偏离其他时钟太远，就要宣告失效，从集群中移除。

#### 有序时间的时间戳

如下所示，在多主复制的数据库中，客户端A在节点1写入数据，客户端B在节点3写入数据，这两个写入都被复制到其他节点。由于时钟偏差，客户端B的写入明明比客户端A的写入要晚，却有更早的时间戳。

多主复制和无主复制的数据库广泛使用最后写入胜利（LWW）的策略来解决冲突。在节点2，由于客户端A的时间戳在后，所以会保留客户端A的写入，而丢弃客户端B的写入。这样，客户端B的写入就丢失了。

![DistributedData ClockProblem](/assets/img/DDIA_DistributedData_ClockProblem.png)

由此可见，LWW的基本问题在于：
- 由于时钟偏差，一些节点的写入会丢失。
- 无法区分频繁的顺序写入（客户端B的递增操作一定发生在客户端A的写入之后）和真正的并发写入（写入者意识不到其他写入者），导致违背因果关系。

即使NTP同步足够准确，也无法避免这种基于时钟的错误排序，因为还有石英钟漂移和网络延迟等影响因素。

像日历时钟和单调钟这样用来测量实际时间的叫做物理时钟。为了对事件进行正确的排序，应该采用基于递增计数器的逻辑时钟，而不是物理时钟。

#### 时钟读数存在置信区间

从机器读取的时间是不准确的，存在误差。因此，时钟读数更像是一段时间范围，例如：以95%的置信度认为当前时间处于本分钟的10.3秒和10.5秒之间。

大多数系统不会公开这种不确定性。Spanner中的Google TrueTime API是个例外，当你询问当前时间时，它会返回可能的最早和最晚时间戳。

#### 全局快照的同步时钟

在快照隔离的实现中，需要单调递增的事务ID。在单节点数据库中，这是很容易实现的，只需要一个计数器。

然而，在分布式数据库中，由于需要协调多个数据中心的多台机器，生成全局单调递增的事务ID比较困难。

Spanner使用同步时钟的时间戳作为事务ID。它使用TrueTime API返回的时钟置信区间，并进行比较：如果两个事务对应的置信区间不重叠，那么很容易判断谁先谁后；如果两个区间重叠，就无法确定事务发生的顺序。

### 进程暂停

在单主数据库中，一个节点怎么知道自己是领导者呢？一种方法是通过租约（lease）。

租约类似于一个带超时的锁，任何一个时刻只有一个节点可以持有租约，成为领导者。在租约到期前，这个节点可以续期，从而继续当领导者。如果这个节点发生故障了，就会停止续期，让另一个节点接管。

这种方法的问题在于，租约的到期时间通常是由另一台机器设置的，如果和本地时钟不同步，可能会出现问题。例如，另一台机器认为租约已经到期，由另一个节点接管了领导者，但当前节点认为租约还没过期，继续处理写入请求。

如果只使用本地单调时钟，会出现另一个问题：如果当前节点在处理请求的过程中，线程暂停了很长时间。在这个期间，租约过期了，另一个节点接管了领导者，但当前节点没有注意到租约过期，还在继续处理请求。

线程暂停是很常见的，发生的原因有：
- 许多编程语言的runtime都有一个垃圾收集器，收集垃圾时需要停止所有正在运行的线程。
- 在虚拟化环境中，可能会挂起虚拟机，即暂停所有进程并把内容保存到磁盘。
- 用户可能会合上笔记本电脑的盖子，导致进程被暂停。
- 当操作系统上下文切换到另一个线程时，当前线程会被暂停。
- 当应用程序同步访问磁盘时，线程可能会暂停，等待缓慢的磁盘I/O操作完成。 

## 知识、真相与谎言

### 真相由多数所定义

在分布式系统中，一个节点看到的情况不一定是真的。例如，对于一个半断开的节点，能够接收所有发送给它的消息，但是无法传出去任何消息，那么即使这个节点运行良好，其他节点也可能会宣告它已经死亡。

许多分布式算法都依赖于法定人数，即在节点之间投票，采取多数的意见作为结果。

#### 分布式锁的一个常见错误

实现分布式锁的时候需要注意：一个节点可能之前拥有过锁，但是过期了却不自知，如果它继续执行一些持有锁才能做的操作，可能会导致错误。

如下所示，客户端1获得租约后，经历了很长的GC pause，期间租约到期了，客户端2获得了租约，并写入数据。等客户端1恢复后，却不知道租约已经到期，继续写入数据，造成文件被破坏。

![DistributedData WrongImplementationOfDistributedClock](/assets/img/DDIA_DistributedData_WrongImplementationOfDistributedClock.png)

#### 防护令牌（fencing token）

当使用锁或租约来保护对某些资源的访问时，需要确保一个误认为自己持有锁或租约的节点不会破坏系统。一种常用的技术是防护令牌。

如下所示，lock service每次在授予锁或租约时，会同时返回一个防护令牌，这个令牌的值是递增的。客户端1获得的防护令牌是33，客户端2获得的防护令牌是34。

要求客户端在写入数据时，必须包含当前的防护令牌。如果存储服务器发现已经有一个更大的令牌写入了，就会拒接当前请求。因此，客户端1的写入被拒绝了，防止了对文件的破坏。

![DistributedData FencingToken](/assets/img/DDIA_DistributedData_FencingToken.png)

### 拜占庭将军问题

防护令牌可以检测和防止无意中发生错误的节点，但是如果有节点想故意破坏系统，完全可以伪造一个防护令牌。

拜占庭故障（Byzantine fault）：节点故意破坏导致的故障。

拜占庭将军问题：如何在不信任的环境中达成共识的问题。

拜占庭容错：系统在一部分节点故意破坏的情况下，仍然能正确工作。

实现拜占庭容错系统的协议很复杂，成本很高，所以一般不会使用。

### 系统模型与现实

关于时序假设，有三种常见的系统模型：
- 同步模型：假设网络延迟、进程暂停、时钟误差永远不会超过某个固定的上限。这个模型并不现实。
- 半同步模型：假设系统大部分时间像同步模型一样运行，偶尔会出现网络延迟、进程暂停、时钟误差超过上限的情况。这个模型比较符合现实。
- 异步模型：不允许对时序有任何假设。

关于节点失效，有三种常见的系统模型：
- 崩溃停止（crash-stop）：节点可能会在任意时刻崩溃，然而永远消失。
- 崩溃恢复（crash-recovery）：节点可能会在任意时刻崩溃，在未知时间之后又可能再次开始响应。这个模型比较符合现实。
- 拜占庭故障：节点可以做任何事情，故意破坏系统。