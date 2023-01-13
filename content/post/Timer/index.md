---
title: "定时器"
date: 2023-01-12
lastmod: 2023-01-12
draft: false
tags: [".NET", "C#"]
categories: [".NET"]
toc: true
---

# 定时器精度

## 缘起...

``` cs
var cts = new CancellationTokenSource();

Task.Run(() =>
{
    for (int i = 0; ; ++i)
    {
        Console.WriteLine(i);
        Thread.Sleep(100);
    }
}, cts.Token);

await Task.Delay(10000);
cts.Cancel();
```

一段很普通的程序, 起一个任务每隔100ms输出一次, 10s之后取消该任务.
按理应该是以输出`100`作为结束, 但是发现程序总是止于`90`(左右).
是标准输出的问题? 去掉标准输出:

``` cs
var cts = new CancellationTokenSource();

int i = 0;

Task.Run(() =>
{
    for (; ; ++i)
    {
        Thread.Sleep(100);
    }
}, cts.Token);

await Task.Delay(10000);
cts.Cancel();
Console.WriteLine(i);
```

问题依旧.

然而同为.NET 7, Linux上则可以正确输出到`100`(左右), 可以看出这是不同系统的
定时器精度的问题了.

[关于Windows定时器精度](https://et-framework.cn/d/245-windowssleep1), Windows
默认设置为`15ms`(60Hz), Linux默认设置为`1ms`.

详见[Windows Timer Resolution: The Great Rule Change](https://randomascii.wordpress.com/2020/10/04/windows-timer-resolution-the-great-rule-change/)

## 设置Windows定时器精度

``` cs
// 常用于多媒体定时器中，与GetTickCount类似，也是返回操作系统启动到现在所经过的毫秒数，精度为1毫秒。
[DllImport("winmm")]
static extern uint timeGetTime();

// 一般默认的精度不止1毫秒（不同操作系统有所不同），需要调用timeBeginPeriod与timeEndPeriod来设置精度
[DllImport("winmm")]
static extern void timeBeginPeriod(int t);
[DllImport("winmm")]
static extern void timeEndPeriod(int t);

// 用法
timeBeginPeriod(1);
uint start = timeGetTime();
Thread.Sleep(2719);
Console.WriteLine(timeGetTime() - start);  //单位毫秒
timeEndPeriod(1);
```
