---
title: AvaloniaUI在TemplatedControl中使用ItemsRepeater
categories:
  - 编程
tags:
  - AvaloniaUI
  - C#
abbrlink: 3354022c
date: 2025-11-22 06:01:33
---

众所周知，AvaloniaUI的`ItemsRepeater`是一个用于显示数据集合的控件，能够很方便地从一个`ObservableCollection`中的数据生成多个子控件。

又众所周知，在AvaloniaUI中，实现自定义控件的最佳实践是使用模板控件（`TemplatedControl`）。

然而，当你想要在`TemplatedControl`中使用`ItemsRepeater`时，你会发现，`ItemsRepeater`内部`DataTemplate`的`DataType`并不会正确使用`ObservableCollection`内部的类。例如，以下这种情况会直接报编译错误：

```xml
<Styles xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:controls="using:MyApp.Controls">

    <Style Selector="controls|ScrollBox">
        <Setter Property="Template">
            <ControlTemplate>
                <ScrollViewer>
                    <ItemsRepeater ItemsSource="{TemplateBinding ItemsSource}">
                        <ItemsRepeater.ItemTemplate>
                            <DataTemplate>
                                <SelectableTextBlock IsEnabled="True" Text="{Binding Line}" Foreground="{Binding LineColor}"/>
                            </DataTemplate>
                        </ItemsRepeater.ItemTemplate>
                    </ItemsRepeater>
                </ScrollViewer>
            </ControlTemplate>
        </Setter>
    </Style>
</Styles>
```

```cs
public class ScrollBox : TemplatedControl
{
    public static readonly StyledProperty<ObservableCollection<ColoredLine>> ItemsSourceProperty =
        AvaloniaProperty.Register<MyTerminal, ObservableCollection<ColoredLine>>(nameof(ItemsSource));
    public ObservableCollection<ColoredLine> ItemsSource
    {
        get => GetValue(ItemsSourceProperty);
        set => SetValue(ItemsSourceProperty, value);
    }
}
```

出现的问题是无法解析某某某类型的数据上下文中的字段或属性 'Line'。

如果你尝试将`DataTemplate`中的`Binding`改为`TemplateBinding`，残念，它的`DataType`会变成整个ScrollBox控件。

## 正解

```xml
<Styles xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:controls="using:MyApp.Controls"
        xmlns:models="using:MyApp.Models"><!--引入ObservableCollection中使用的数据类型-->

    <Style Selector="controls|ScrollBox">
        <Setter Property="Template">
            <ControlTemplate>
                <ScrollViewer>
                    <ItemsRepeater ItemsSource="{TemplateBinding ItemsSource}">
                        <ItemsRepeater.ItemTemplate>
                            <DataTemplate DataType="models:ColoredLine"><!--手动设置DataType-->
                                <SelectableTextBlock IsEnabled="True" Text="{Binding Line}" Foreground="{Binding LineColor}"/>
                            </DataTemplate>
                        </ItemsRepeater.ItemTemplate>
                    </ItemsRepeater>
                </ScrollViewer>
            </ControlTemplate>
        </Setter>
    </Style>
</Styles>
```

与直接在实例控件中使用`ItemsRepeater`不同，在`TemplatedControl`中使用`ItemsRepeater`时，分析器无法直接确定`DataTemplate`的类型，需要手动指定`DataType`。

为了应用程序的性能，我个人是一直开着编译时绑定的，不知道不使用编译时绑定的应用有没有这个问题。