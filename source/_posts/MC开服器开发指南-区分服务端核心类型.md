---
title: MC开服器开发指南-区分服务端核心类型
date: 2025-02-12T19:35:51+08:00
categories: 
  - 编程
tags:
  - Minecraft
  - Minecraft开服器
  - C#
  - Java
---
简而言之，就是检查.jar文件内的{% span code::META-INF/MANIFEST.MF %}文件，其中的Main-Class字段具有每种核心的特征。

## 引言

在开发开服器时，区分服务端类型是很重要的，因为单凭文件扩展名无法判断用户提供的.jar文件究竟是什么，例如反人类的Forge核心，永远不给下载核心本体，只让下载安装器。除此之外，如果客户端核心滥竽充数，也会导致某些奇怪的问题。

然而，有一个办法可以大致上区分各种服务器核心，那就是检查.jar文件内的{% span code::META-INF/MANIFEST.MF %}文件。

{% folding cyan::关于MAINFEST.MF %}

{% span code::MANIFEST.MF %}文件是Java平台的一种规范，用于定义和管理Java应用程序的组件、库和模块。它是JAR文件中的一个纯文本文件，遵循特定的格式规范。

在JAR文件中，{% span code::MANIFEST.MF %}文件必须位于META-INF目录下，且一个JAR文件中只能有一个{% span code::MANIFEST.MF %}文件。

{% endfolding %}

在MANIFEST.MF文件中，Main-Class字段向Java虚拟机指明了该文件的主类，以使jar文件能够正常执行。在较新的MC版本中都会具有Main-Class字段（目前已知的只有远古版本没有）。由于不同的核心都会有自己的主类名称，因此可以通过这个特征在MC开服器中分辨其核心类型。

``` csharp
public class CoreValidationService
{
    public enum CoreType// 核心类型的枚举
    {
        Error,
        Unknown,
        Client,
        ForgeInstaller,
        FabricInstaller,
        Forge,
        Fabric,
        Arclight,
        CatServer,
        CraftBukkit,
        Leaves,
        LightFall,
        Mohist,
        Paper,
        Vanilla,
        Velocity,
    }
    public static CoreType Validate(string? filePath, out string ErrorMessage)// 校验核心类型
    {
        ErrorMessage = "";
        if (string.IsNullOrEmpty(filePath))
        {
            ErrorMessage = "选定的路径为空";
            return CoreType.Error;
        }
        if (!File.Exists(filePath))
        {
            ErrorMessage = "选定的文件/路径不存在";
            return CoreType.Error;
        }
        string? JarMainClass = GetMainClass(filePath);
        if (JarMainClass == null) return CoreType.Unknown;
        else if (JarMainClass.StartsWith("Access denied") || JarMainClass.StartsWith("Error"))
        {
            ErrorMessage = JarMainClass;
            return CoreType.Error;
        }
        else
        {
            return JarMainClass switch
            {
                "net.minecraft.server.MinecraftServer" => CoreType.Vanilla,
                "net.minecraft.bundler.Main" => CoreType.Vanilla,
                "net.minecraft.client.Main" => CoreType.Client,
                "net.minecraftforge.installer.SimpleInstaller" => CoreType.ForgeInstaller,
                "net.fabricmc.installer.Main" => CoreType.FabricInstaller,
                "net.fabricmc.installer.ServerLauncher" => CoreType.Fabric,
                "io.izzel.arclight.server.Launcher" => CoreType.Arclight,
                "catserver.server.CatServerLaunch" => CoreType.CatServer,
                "foxlaunch.FoxServerLauncher" => CoreType.CatServer,
                "org.bukkit.craftbukkit.Main" => CoreType.CraftBukkit,
                "org.bukkit.craftbukkit.bootstrap.Main" => CoreType.CraftBukkit,
                "io.papermc.paperclip.Main" => CoreType.Paper,
                "org.leavesmc.leavesclip.Main" => CoreType.Leaves,
                "net.md_5.bungee.Bootstrap" => CoreType.LightFall,
                "com.mohistmc.MohistMCStart" => CoreType.Mohist,
                "com.mohistmc.MohistMC" => CoreType.Mohist,
                "com.destroystokyo.paperclip.Paperclip" => CoreType.Paper,
                "com.velocitypowered.proxy.Velocity" => CoreType.Velocity,
                _ => CoreType.Unknown,
            };
        }
    }
    // the following code is taken from https://github.com/Orange-Icepop/JavaMainClassFinder
    public static string? GetMainClass(string jarFilePath)
    {
        try
        {
            using FileStream stream = new FileStream(jarFilePath, FileMode.Open);
            using ZipArchive archive = new ZipArchive(stream, ZipArchiveMode.Read);
            ZipArchiveEntry? manifestEntry = archive.Entries.FirstOrDefault(entry => entry.FullName == "META-INF/MANIFEST.MF");
            if (manifestEntry != null)// 对于较新版本的MC，MANIFEST.MF中应当包含Main-Class字段
            {
                using StreamReader reader = new StreamReader(manifestEntry.Open());
                string manifestContent = reader.ReadToEnd();
                return FindMainClassLine(manifestContent);
            }
            return null;
        }
        catch (UnauthorizedAccessException ex)
        {
            Console.WriteLine("Access denied: " + ex.Message);
            return "Access denied: " + ex.Message;
        }
        catch (IOException ex)
        {
            Console.WriteLine("IO error: " + ex.Message);
            return "Error reading file: " + ex.Message;
        }
        catch (Exception ex)
        {
            Console.WriteLine("Error reading jar file: " + ex.Message);
            return "Error reading jar file: " + ex.Message;
        }
    }

    public static string? FindMainClassLine(string manifestContent)
    {
        string[] lines = manifestContent.Split(["\r\n", "\r", "\n"], StringSplitOptions.None);
        foreach (string line in lines)
        {
            if (line.StartsWith("Main-Class:"))
            {
                return line.Substring("Main-Class:".Length).Trim();
            }
        }
        return null;
    }
}
```

最重要的信息是以下这段：

``` csharp
return JarMainClass switch//使用switch表达式返回值
{
    "net.minecraft.server.MinecraftServer" => CoreType.Vanilla,
    "net.minecraft.bundler.Main" => CoreType.Vanilla,
    "net.minecraft.client.Main" => CoreType.Client,
    "net.minecraftforge.installer.SimpleInstaller" => CoreType.ForgeInstaller,
    "net.fabricmc.installer.Main" => CoreType.FabricInstaller,
    "net.fabricmc.installer.ServerLauncher" => CoreType.Fabric,
    "io.izzel.arclight.server.Launcher" => CoreType.Arclight,
    "catserver.server.CatServerLaunch" => CoreType.CatServer,
    "foxlaunch.FoxServerLauncher" => CoreType.CatServer,
    "org.bukkit.craftbukkit.Main" => CoreType.CraftBukkit,
    "org.bukkit.craftbukkit.bootstrap.Main" => CoreType.CraftBukkit,
    "io.papermc.paperclip.Main" => CoreType.Paper,
    "org.leavesmc.leavesclip.Main" => CoreType.Leaves,
    "net.md_5.bungee.Bootstrap" => CoreType.LightFall,
    "com.mohistmc.MohistMCStart" => CoreType.Mohist,
    "com.mohistmc.MohistMC" => CoreType.Mohist,
    "com.destroystokyo.paperclip.Paperclip" => CoreType.Paper,
    "com.velocitypowered.proxy.Velocity" => CoreType.Velocity,
    _ => CoreType.Unknown,
};
```

比较需要注意的是，某些核心在不同的版本中会具有不同的主类名称，例如原版核心的最近几个版本（实测最晚为1.21）的主类名称从{% span code::net.minecraft.server.MinecraftServer %}换为了{% span code::net.minecraft.bundler.Main %}，但是客户端的主类没改。

还有一部分核心由于底层相同，并没有修改主类名称，例如Folia和Paper的主类名称就相同。

## 部分核心的问题

之前的代码要特意标注来自LSL v0.08是有原因的。在0.08之后，我对所有类型的核心进行了测试，发现Mohist核心与Folia核心会在解析时抛出一大堆的ArgumentOutOfRangeException异常。经过查询，这是在解析时间戳时出现的问题，非常有可能是因为这些核心的时间戳异常导致的。当然，这不影响核心文件的执行，但是对于上面我们依赖的System.IO.Compression命名空间中的方法具有致命的执行效率打击，也就是在0.08版本和0.07.1版本中添加某些核心时会导致整个应用程序卡死的原因。

为了解决这个问题，我在0.08.1版本中引入了{% span code::SharpZipLib %}库，这个库不会理会时间戳异常，因此完美规避掉了这个问题。完整实现代码如下：

``` csharp
using ICSharpCode.SharpZipLib.Zip;
using System;
using System.IO;
 
namespace LSL.Services.Validators
{
    public class CoreValidationService
    {
        public enum CoreType
        {
            Error,
            Unknown,
            Client,
            ForgeInstaller,
            FabricInstaller,
            Forge,
            Fabric,
            Arclight,
            CatServer,
            CraftBukkit,
            Leaves,
            LightFall,
            Mohist,
            Paper,
            Vanilla,
            Velocity,
        }
        public static CoreType Validate(string? filePath, out string ErrorMessage)// 校验核心类型
        {
            ErrorMessage = "";
            if (string.IsNullOrEmpty(filePath))
            {
                ErrorMessage = "选定的路径为空";
                return CoreType.Error;
 
            }
            if (!File.Exists(filePath))
            {
                ErrorMessage = "选定的文件/路径不存在";
                return CoreType.Error;
            }
            string? JarMainClass = GetMainClass(filePath);
            if (JarMainClass == null) return CoreType.Unknown;
            else if (JarMainClass.StartsWith("Access denied") || JarMainClass.StartsWith("Error"))
            {
                ErrorMessage = JarMainClass;
                return CoreType.Error;
            }
            else
            {
                return JarMainClass switch
                {
                    "net.minecraft.server.MinecraftServer" => CoreType.Vanilla,
                    "net.minecraft.bundler.Main" => CoreType.Vanilla,
                    "net.minecraft.client.Main" => CoreType.Client,
                    "net.minecraftforge.installer.SimpleInstaller" => CoreType.ForgeInstaller,
                    "net.fabricmc.installer.Main" => CoreType.FabricInstaller,
                    "net.fabricmc.installer.ServerLauncher" => CoreType.Fabric,
                    "io.izzel.arclight.server.Launcher" => CoreType.Arclight,
                    "catserver.server.CatServerLaunch" => CoreType.CatServer,
                    "foxlaunch.FoxServerLauncher" => CoreType.CatServer,
                    "org.bukkit.craftbukkit.Main" => CoreType.CraftBukkit,
                    "org.bukkit.craftbukkit.bootstrap.Main" => CoreType.CraftBukkit,
                    "io.papermc.paperclip.Main" => CoreType.Paper,
                    "org.leavesmc.leavesclip.Main" => CoreType.Leaves,
                    "net.md_5.bungee.Bootstrap" => CoreType.LightFall,
                    "com.mohistmc.MohistMCStart" => CoreType.Mohist,
                    "com.mohistmc.MohistMC" => CoreType.Mohist,
                    "com.destroystokyo.paperclip.Paperclip" => CoreType.Paper,
                    "com.velocitypowered.proxy.Velocity" => CoreType.Velocity,
                    _ => CoreType.Unknown,
                };
            }
        }
        // the following code is taken from https://github.com/Orange-Icepop/JavaMainClassFinder
        public static string? GetMainClass(string jarFilePath)
        {
            try
            {
                using FileStream stream = new FileStream(jarFilePath, FileMode.Open);
                using (ZipFile zipFile = new ZipFile(stream))
                {
                    foreach (ZipEntry entry in zipFile)
                    {
                        if (entry.IsDirectory)
                            continue;
 
                        if (entry.Name == "META-INF/MANIFEST.MF")// 对于较新版本的MC，MANIFEST.MF中应当包含Main-Class字段
                        {
                            using (var fstream = zipFile.GetInputStream(entry))
                            using (StreamReader reader = new StreamReader(fstream))
                            {
                                string manifestContent = reader.ReadToEnd();
                                return FindMainClassLine(manifestContent);
                            }
                        }
                    }
                }
                return null;
            }
            catch (UnauthorizedAccessException ex)
            {
                Console.WriteLine("Access denied: " + ex.Message);
                return "Access denied: " + ex.Message;
            }
            catch (IOException ex)
            {
                Console.WriteLine("IO error: " + ex.Message);
                return "Error reading file: " + ex.Message;
            }
            catch (Exception ex)
            {
                Console.WriteLine("Error reading jar file: " + ex.Message);
                return "Error reading jar file: " + ex.Message;
            }
        }
 
        public static string? FindMainClassLine(string manifestContent)
        {
            string[] lines = manifestContent.Split(["\r\n", "\r", "\n"], StringSplitOptions.None);
            foreach (string line in lines)
            {
                if (line.StartsWith("Main-Class:"))
                {
                    return line.Substring("Main-Class:".Length).Trim();
                }
            }
            return null;
        }
    }
}
```

[JavaMainClassFinder](https://github.com/Orange-Icepop/JavaMainClassFinder)也同步进行了更新。问题解决！