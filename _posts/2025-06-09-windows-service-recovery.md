---
layout: post
title: 如何配置Windows Service的崩溃恢复策略？
date: 2025-06-09 +0800
tags: [Windows, 崩溃恢复]
categories: [Windows系统]
---

我们组有一个服务是作为Windows Service部署的，在服务崩溃后，希望能够借助Windows系统的一些机制来自动重启。最近对这个问题进行了一点研究，在此记录一下。

## 设置服务的Startup Type

在创建Windows Service的时候，可以设置`StartupType`，指定在操作系统启动后，希望用哪种方式来启动服务。`StartupType`的值可以是：
- `Automatic`：在操作系统启动后，尽快自动启动服务。
- `AutomaticDelayedStart`：在操作系统启动后，延迟一段时间（不固定）后自动启动服务。
- `Manual`：在操作系统启动后，手动启动服务。
- `Disabled`：服务不可以被启动。

### 对比`Automatic`和`AutomaticDelayedStart`

`Automatic`的优先级高于`AutomaticDelayedStart`。在系统启动初期，Service Control Manager（SCM）会优先启动所有`Automatic`服务。在系统更加稳定后，SCM才会去启动所有`AutomaticDelayedStart`服务。

`Automatic`服务会影响开机速度，而`AutomaticDelayedStart`不会影响。

关键服务选`Automatic`，非关键、重资源的服务选`AutomaticDelayedStart`。

## 设置服务的崩溃恢复策略

设置服务的`StartupType`只能影响系统启动后的启动状态，如果服务在运行过程中突然崩溃了，需要另外设置崩溃恢复策略来重启服务。

### 命令

设置崩溃恢复策略的命令如下：
```cmd
sc.exe failure <ServiceName> reset= <ResetPeriodInSeconds> actions= <action1>/<delay1InMs>/<action2>/<delay2InMs>/... reboot= <RebootMessage> command= <CommandLine>
```

其中，
- `actions=`表示崩溃后采取的恢复措施。
    - 支持的action类型：
        - restart：重启服务
        - run：运行指定程序（需配合`command=`)
        - reboot：重启机器（需管理员权限）
        - none（或任意其他字符串）：不执行任何操作
    - `<delayXInMs>`表示延迟多久执行action。
    - 崩溃次数超过action个数后，会一直按照最后一个action来执行，即最后一个action会被无限重复执行。
- `reset=`表示多久后重置崩溃次数。
    - 从最近一次崩溃开始计时，而不是从第一次崩溃开始计时。

查看崩溃恢复策略的命令如下：
```cmd
sc.exe qfailure <ServiceName>
```

### 触发条件

手动停止服务是不会触发崩溃恢复的，只有服务意外崩溃时才会触发，包括以下情况：
- 服务的运行过程中抛出未捕获的异常，导致进程崩溃。
- 服务返回非0退出码，如返回`Environment.Exit(1)`。
- 进程被强制kill掉，如使用命令`taskkill /f /im <ProcessName>`。

### 查看event log

可以在`Windows Logs -> System`中查看崩溃恢复的日志，`Level`为`Error`，`Source`为`Service Control Manager`。

### 例子

#### 例1：无限次重启

```
>> sc.exe failure TestWindowsService reset= 10 actions= restart/0/restart/3000
[SC] ChangeServiceConfig2 SUCCESS

>> sc.exe qfailure TestWindowsService
[SC] QueryServiceConfig2 SUCCESS

SERVICE_NAME: TestWindowsService
        RESET_PERIOD (in seconds)    : 10
        REBOOT_MESSAGE               :
        COMMAND_LINE                 :
        FAILURE_ACTIONS              : RESTART -- Delay = 0 milliseconds.
                                       RESTART -- Delay = 3000 milliseconds.
```

- 第一次崩溃：立刻重启
    ```
    The Test Windows Service service terminated unexpectedly.  It has done this 1 time(s).  The following corrective action will be taken in 0 milliseconds: Restart the service.
    ```
- 第二次崩溃：3s后重启
    ```
    The Test Windows Service service terminated unexpectedly.  It has done this 2 time(s).  The following corrective action will be taken in 3000 milliseconds: Restart the service.
    ```
- 第三次崩溃：3s后重启
    ```
    The Test Windows Service service terminated unexpectedly.  It has done this 3 time(s).  The following corrective action will be taken in 3000 milliseconds: Restart the service.
    ```
- 距离第三次崩溃10s后，崩溃次数被重置
- 第一次崩溃：立刻重启
    ```
    The Test Windows Service service terminated unexpectedly.  It has done this 1 time(s).  The following corrective action will be taken in 0 milliseconds: Restart the service.
    ```

#### 例2：限制重启次数

```
>> sc.exe failure TestWindowsService reset= 10 actions= restart/0/restart/3000/none/0
[SC] ChangeServiceConfig2 SUCCESS

>> sc.exe qfailure TestWindowsService
[SC] QueryServiceConfig2 SUCCESS

SERVICE_NAME: TestWindowsService
        RESET_PERIOD (in seconds)    : 10
        REBOOT_MESSAGE               :
        COMMAND_LINE                 :
        FAILURE_ACTIONS              : RESTART -- Delay = 0 milliseconds.
                                       RESTART -- Delay = 3000 milliseconds.
```

- 第一次崩溃：立刻重启
    ```
    The Test Windows Service service terminated unexpectedly.  It has done this 1 time(s).  The following corrective action will be taken in 0 milliseconds: Restart the service.
    ```
- 第二次崩溃：3s后重启
    ```
    The Test Windows Service service terminated unexpectedly.  It has done this 2 time(s).  The following corrective action will be taken in 3000 milliseconds: Restart the service.
    ```
- 第三次崩溃：不重启
    ```
    The Test Windows Service service terminated unexpectedly.  It has done this 3 time(s).
    ```
- 距离第三次崩溃10s后：不重启

#### 例3：设置`reset=`为0

```
>> sc.exe failure TestWindowsService reset= 0 actions= restart/0/restart/3000/none/0
[SC] ChangeServiceConfig2 SUCCESS

>> sc.exe qfailure TestWindowsService
[SC] QueryServiceConfig2 SUCCESS

SERVICE_NAME: TestWindowsService
        RESET_PERIOD (in seconds)    : 0
        REBOOT_MESSAGE               :
        COMMAND_LINE                 :
        FAILURE_ACTIONS              : RESTART -- Delay = 0 milliseconds.
                                       RESTART -- Delay = 3000 milliseconds.
```

- 第一次崩溃：立马重启，崩溃次数立马被重置
    ```
    The Test Windows Service service terminated unexpectedly.  It has done this 1 time(s).  The following corrective action will be taken in 0 milliseconds: Restart the service.
    ```
- 第一次崩溃：立马重启，崩溃次数立马被重启
    ```
    The Test Windows Service service terminated unexpectedly.  It has done this 1 time(s).  The following corrective action will be taken in 0 milliseconds: Restart the service.
    ```

## 参考
- [sc failure](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc742019(v=ws.11))
- [sc qfailure](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc742047(v=ws.11))