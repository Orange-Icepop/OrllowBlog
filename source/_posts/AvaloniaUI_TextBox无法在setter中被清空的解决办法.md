---
title: AvaloniaUI TextBox无法在setter中被清空的解决办法
date: 2025-08-06T21:24:34+08:00
categories: 
  - 编程
tags: 
  - AvaloniaUI
  - .NET
  - C#
---

先上代码：

```cs
private string _inputText = "";
public string InputText
{
    get => _inputText;
    set
    {
        string endless = value.TrimEnd('\r', '\n');
        if (endless.Length < value.Length)
        {
            this.RaiseAndSetIfChanged(ref _inputText, "");
        }
        else 
        {
            this.RaiseAndSetIfChanged(ref _inputText, value); 
        }
    }
}
```

在这段C#代码中，`InputText`访问器使用ReactiveUI双向绑定到视图的一个TextBox中，当在TextBox中输入回车后并没有像预期的那样被清空。

按照常规逻辑来理解，当用户回车时，在setter中会将空字符串赋值给`_inputText`字段并触发属性更新，照理来说应该更新为空值才对。但是，经过实际测验，`_inputText`字段仍然保留带回车的原始值。

这一现象的核心原因是，set块的执行与对`_inputText`字段的修改位于同一个同步调用栈中，而ReactiveUI等框架会自动合并属性通知到setter本身，导致setter内的其它属性通知无效化。

解决方案就是将更新操作丢出去。具体来说，就是在AvaloniaUI中使用`Dispatcher.UIThread.Post(()=>this.RaiseAndSetIfChanged(ref _inputText, ""))`而不是直接触发通知操作。这样，更新操作就被延后到了UI线程处理队列中，不再受属性通知合并的影响。

此方法本质是通过操作时序的重排规避了框架的优化机制，是一种“以空间换时间”的解决方案。