---
title: C#获取Process的性能占用信息
date: 2025-12-11T20:49:13+08:00
categories: 
  - 编程
tags:
  - C#
  - .NET
---

很多MC服务器启动器高级功能中有一项是能够实时监控服务端进程的性能占用，这需要管理器进程能够获取到服务端进程的CPU和内存占用情况。

[LSL](https://github.com/Orange-Icepop/LSL)是使用C#编写的，但是.NET没有任何现成的类能够集中监控这些数据。唯一一个能够获取到这些数据的类是.NET Framework 4.6.2中的`PerformanceCounter`，但是首先，这玩意儿在高版本的.NET中已经被移除了；其次，它只兼容Windows平台；第三，这东西虽然详细，但是没有任何卵用，因为它监控的是C#应用程序本身的性能信息，管不到`Process`对象上......

所以，我们只能自己动手丰衣足食了。

## 示例

```cs
using System;
using System.Diagnostics;
using System.Threading;

namespace LSL.Services.ServerServices;

/// <summary>
/// A metrics monitor attached to a Process instance.
/// </summary>
public class ProcessMetricsMonitor : IDisposable
{
    // 用于上报进程性能数据的EventHandler
    public event EventHandler<ProcessMetricsEventArgs>? MetricsUpdated;
    private readonly Timer _timer;// 用于定时上报进程性能数据的Timer
    private readonly Process _process;// 要监控的进程
    private readonly long _allocatedMemoryBytes;// 进程分配的内存大小，用于直接计算内存占用率
    private TimeSpan _prevCpuTime;// 上一次获取到的CPU占用时间
    private DateTime _prevTime;// 上一次获取信息的时间戳
    private readonly object _lock = new();
    private bool _disposed;

    public ProcessMetricsMonitor(Process process, long allocatedMemoryBytes, int interval = 1000)
    {
        _process = process;
        _allocatedMemoryBytes = allocatedMemoryBytes;
        _prevCpuTime = process.TotalProcessorTime;
        _prevTime = DateTime.UtcNow;
        
        // 创建定时器（首次触发在1秒后，之后每秒触发）
        _timer = new Timer(OnTimerCallback, null, interval, interval);
    }

    private void OnTimerCallback(object? state)
    {
        lock (_lock)// 确保线程安全，避免上一次获取信息和本次获取信息之间的竞争（虽然概率很小，但是如果CPU忙的话就说不定了...还是保险一点）
        {
            if (_disposed) return;// 如果服务器进程已经退出，则不再执行

            double cpuUsage = 0;
            long processMemory = 0;
            bool isExited;

            try
            {
                // 检查进程是否已退出
                isExited = _process.HasExited;
                
                if (!isExited)
                {
                    _process.Refresh();
                    // 计算CPU使用率
                    var currentTime = DateTime.UtcNow;
                    var currentCpuTime = _process.TotalProcessorTime;
                    
                    var cpuElapsed = (currentCpuTime - _prevCpuTime).TotalSeconds;
                    var timeElapsed = (currentTime - _prevTime).TotalSeconds;
                    
                    _prevTime = currentTime;
                    _prevCpuTime = currentCpuTime;

                    if (timeElapsed > 0 && cpuElapsed > 0)
                    {
                        cpuUsage = (cpuElapsed / timeElapsed) * 100;
                        cpuUsage /= Environment.ProcessorCount; // 多核百分比
                    }

                    // 计算内存使用量
                    processMemory = _process.PrivateMemorySize64;
                }
            }
            catch (InvalidOperationException)
            {
                // 进程已退出或无法访问
                isExited = true;
            }
            catch (Exception ex)
            {
                // 触发包含错误信息的事件
                MetricsUpdated?.Invoke(this, new ProcessMetricsEventArgs(_id, 0, 0, 0, true, ex.Message));
                return;
            }

            // 触发事件（即使进程已退出也通知）
            MetricsUpdated?.Invoke(this, new ProcessMetricsEventArgs(
                cpuUsage, 
                processMemory, 
                _allocatedMemoryBytes,
                isExited
            ));
        }
    }
    public void Dispose()
    {
        lock (_lock)
        {
            if (_disposed) return;
            
            MetricsUpdated?.Invoke(this, new ProcessMetricsEventArgs(
                0, 
                0, 
                _allocatedMemoryBytes,
                true
            ));
            _timer.Dispose();
            _disposed = true;
            GC.SuppressFinalize(this);
        }
    }
}
```

## 详解

我个人对这个实现比较满意，它没有对`Process`类本身进行修改或者使用附加方法（.NET里面涉及到非托管资源的还是得慎重），而是让监控器持有一个`Process`对象，通过定时器定时获取进程的CPU使用率、内存占用、分配内存等信息，然后通过事件通知给外部。这样，外部只需要订阅事件，就可以获取到进程的性能信息了。

### 1. 获取CPU使用率

CPU占用率没有很好的指标，目前能想到的只有`Process`自带的`TotalProcessorTime`属性，它表示进程自启动以来所消耗的CPU时间，因此我们可以通过计算两次获取的`TotalProcessorTime`的差值，除以CPU核数（可以在一个全局的static class中缓存以减少对Environment类的调用开销），再除以两次获取的时间间隔，得到CPU使用率。这是一个笨办法，但能解决问题。唯一的缺点就是无法精确到单个线程，只能得到进程整体的CPU使用率。

{%note info::关于为什么不直接除以一秒，是由于Timer本身就不是为高精度计时设计的，每次CallBack的时间间隔都有微小差异，当CPU忙的时候甚至有可能多个任务堆积导致极大的误差，因此我选择了采用TimeSpan计时的方式。%}

### 2. 获取内存占用

内存占用就相当简单了，可以直接通过`Process`自带的`PrivateMemorySize64`属性获取，它表示进程独占的内存提交量大小（不要使用`WorkingSet64`，它代表的是物理内存中的活跃部分，和程序占用的的总内存大小有本质差异）。我还在构造函数中直接引入了配置中给它分配的内存字节数，这样传出的就是两个表示占用的浮点数，格式较为统一。

这个方案有一个无法克服的缺点，就是给JVM传递的参数是JVM堆内存的限制，而JVM实际使用的内存不止这点，因此这个方案有概率使得内存占用超过100%，而且分配的堆内存越小出现这种情况的概率越大。想要破解这个问题只能在JVM上动手脚，但是这种侵入式的设计我不是很想干...就暂时这样了。

除此之外，还有一个容易遗漏的重点，就是用于刷新进程状态的`_process.Refresh()`方法的调用。该方法对于我们的实时监测非常重要，因为`PrivateMemorySize64`属性不会实时更新，如果不调用，性能监控图上就会是一条直线，因为获取到的内存占用值始终是同一个值。