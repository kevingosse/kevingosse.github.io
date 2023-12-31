---
url: writing-clrmd-extensions-for-windbg-and-lldb-916427956f66
canonical_url: https://medium.com/criteo-engineering/writing-clrmd-extensions-for-windbg-and-lldb-916427956f66
title: Writing ClrMD extensions for WinDbg and LLDB
subtitle: Guide on how to easily write debugger extensions that are compatible with
  both WinDbg and LLDB
date: 2019-03-14
description: Guide on how to easily write debugger extensions that are compatible
  with both WinDbg and LLDB
tags:
- dotnet
- debugging
- clrmd
- lldb
- windbg
author: Kevin Gosse
thumbnailImage: /images/2019-03-14-writing-clrmd-extensions-for-windbg-and-lldb-916427956f66-1.webp
---

You may have already read [the CriteoLabs article about how to write ClrMD extensions for WinDbg](https://labs.criteo.com/2017/06/clrmd-part-5-how-to-use-clrmd-to-extend-sos-in-windbg/). As we move to Linux, we realized that we could not use our debugging toolbox anymore as it was written for WinDbg. Since LLDB is the common debugger for .net NET Core on Linux, I decided to write a compatibility layer to be able to load our extensions in the new environment. And while I was at it, I tried to make the overall process of writing such a debugger extension a bit simpler.

# Introducing ClrMDExports

How to create an extension that will work with both WinDbg and LLDB? The first step is still to create a new Class Library project. Both .NET Framework and .NET Standard are supported, so feel free to use the one you prefer. Some things to note though:

* If you choose .NET Framework, make sure not to use any feature that is not compatible with .NET Core (such as AppDomain), or you won't be able to run your extension in LLDB

* If you choose .NET Standard, remember to publish your project to have all the dependencies contained in one folder, as this is not done by default when you compile

After creating your project, add a reference to [the ClrMDExports nuget package](https://www.nuget.org/packages/ClrMDExports/). It will automatically pull ClrMD and [UnmanagedExports.Repack](https://www.nuget.org/packages/UnmanagedExports.Repack/) as dependencies. UnmanagedExports.Repack is a fork of UnmanagedExports which adds compatibility with .NET Framework 4.7+ and .NET Standard, and supports PackageReference. **Make sure that your project targets x86 or x64, as UnmanagedExports won't work with AnyCPU.**

Note that a new *Init.cs* file gets added to your project (it should not be visible if you use package references). **Do not make any change to this file.** It will be overwritten each time you update the nuget package.

The Init file takes care of exporting the `DebugExtensionInitialize` method needed by WinDbg, and setups everything so that dependencies are correctly loaded as long as they're in the same folder as your extension.

The next step is to add your custom commands. You need to create one static method for each command, with the following signature:

```csharp
public static void HelloWorld(IntPtr client, [MarshalAs(UnmanagedType.LPStr)] string args)
{
}
```

Then decorate it with the `DllExport` attribute that comes with UnmanagedExports. You can use the `ExportName` parameter of the attribute to define the command name that will be visible to WinDbg/LLDB. Remember that names are case-sensitive!

```csharp
[DllExport("helloworld")]
public static void HelloWorld(IntPtr client, [MarshalAs(UnmanagedType.LPStr)] string args)
{
}
```

In that method, you should only call the method `DebuggingContext.Execute` provided by ClrMDExports. It takes the value of `client` and `args` as parameters, as well as a delegate to another static method with the `(ClrRuntime runtime, string args)` signature. It's in that static callback method that you will implement the logic of your command.

```csharp
[DllExport("helloworld")]
public static void HelloWorld(IntPtr client, [MarshalAs(UnmanagedType.LPStr)] string args)
{
    DebuggingContext.Execute(client, args, HelloWorld);
}

private static void HelloWorld(ClrRuntime runtime, string args)
{
    Console.WriteLine("The first 10 types on the heap are: ");

    foreach (var type in runtime.Heap.EnumerateTypes().Take(10))
    {
        Console.WriteLine(type);
    }
}
```

For your convenience, the console output is automatically redirected to the debugger.

And that's it! From there, you can directly load and use your extension in WinDbg:

{{<image classes="fancybox" src="/images/2019-03-14-writing-clrmd-extensions-for-windbg-and-lldb-916427956f66-1.webp" title="Writing a ClrMD extension for WinDbg is now a matter of minutes" >}}

# Running in LLDB on Linux

As the extension is written for the WinDbg API, it cannot be directly loaded into LLDB. Instead, [I wrote a meta-plugin](https://github.com/kevingosse/LLDB-LoadManaged/) that does the translation.

How to use it? First, [download the latest release](https://github.com/kevingosse/LLDB-LoadManaged/releases) of the LLDB-LoadManaged meta-plugin and unzip it in a folder.

Then start LLDB and attach to a target (live process or crash dump):

```bash
$ ./lldb -c dump.dmp
```

Next, load the meta-plugin:

```bash
plugin load ./loadmanaged/libloadmanaged.so
```

It's important to make sure that the *Mono.Cecil.dll* and *PluginInterop.dll* files are located in the same folder as *libloadmanaged.so*.

Upon load, LLDB-LoadManaged will try to locate CoreCLR by browsing the modules loaded in your debug target. If it fails (for instance, because you're running lldb on a different machine than the target), you can manually set the path by calling `SetClrPath`:

```bash
SetClrPath /usr/local/share/dotnet/shared/Microsoft.NETCore.App/2.2.0/
```

Lastly, load the WinDbg extension using the `LoadManaged` command:

```bash
LoadManaged /home/k.gosse/TestExtension.dll
```

(the `LoadManaged` command does not support relative paths yet)

And that's it! Now you can call the commands exposed by the extension just like you would in WinDbg.

{{<image classes="fancybox" src="/images/2019-03-14-writing-clrmd-extensions-for-windbg-and-lldb-916427956f66-2.webp" title="The ClrMD extension for WinDbg runs flawlessly in LLDB on Linux" >}}

Note: both *libloadmanaged.so* and *libsosplugin.so* host a CLR for their own needs. Unfortunately, [the .NET Core CLR does not support side-by-side scenarios](https://github.com/dotnet/coreclr/issues/22529). It means you cannot use LoadManaged and the SOS plugin at the same time. I'm aware this is a huge limitation, but it's unlikely that it gets fixed on .NET Core side. As a workaround, I will probably work on a managed version of SOS that can be loaded through LoadManaged and replace *libsosplugin.so*.

# What's next

This is still a very early version of LLDB-LoadManaged. In the coming weeks, I'd like to improve the error handling and make the CLR path detection smarter. Still, we already use it on a regular basis at Criteo, so it should be stable enough for the common use-cases. The main added value of using LLDB versus a standalone ClrMD applications is the possibility to attach to a live process (ClrMD on Linux does not support that yet). I also know there's been some work on a cross-platform REPL environment based on ClrMD ([https://github.com/dotnet/diagnostics/tree/master/src/Tools](https://github.com/dotnet/diagnostics/tree/master/src/Tools)), so it will be nice to see how both efforts can converge.
