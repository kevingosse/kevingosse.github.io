---
url: writing-native-windbg-extensions-in-c-5390726f3cec
canonical_url: https://medium.com/@kevingosse/writing-native-windbg-extensions-in-c-5390726f3cec
title: Writing native WinDbg extensions in C#
subtitle: Using NativeAOT and ClrMD to write a native WinDbg extension in .NET.
summary: Using NativeAOT and ClrMD to write a native WinDbg extension in .NET.
date: 2022-02-08
description: ""
tags:
- dotnet
- windbg
- debugging
- nativeaot
- clrmd
author: Kevin Gosse
thumbnailImage: /images/writing-native-windbg-extensions-in-c-5390726f3cec-1.webp
---

# Writing native WinDbg extensions in C#

It has already been possible for a long time to write WinDbg extensions in C#, for instance using ClrMD [as described by Christophe Nasarre in this article](https://labs.criteo.com/2017/06/clrmd-part-5-how-to-use-clrmd-to-extend-sos-in-windbg/). However, it has a few serious drawbacks:

* Dependencies are tricky to manage, unless you store all the extension files in the same folder as WinDbg

* Because of the tricks used to manage dependencies, you can run into conflicts if you simultaneously load multiple extensions written in C#

* Exporting native methods is not directly supported so you need to use nuget packages such as [UnmanagedExports](https://www.nuget.org/packages/UnmanagedExports/) (that relies on decompiling/recompiling the assemblies using ildasm) and they can break in unexpected ways

* It was only tested with .NET Framework, I'm not sure it's possible to write extensions in .NET Core (unless you use [the CLR hosting API](https://docs.microsoft.com/en-us/dotnet/core/tutorials/netcore-hosting?WT.mc_id=DT-MVP-5003493))

I tried to fix a few of those points in my [ClrMDExports](https://www.nuget.org/packages/ClrMDExports/) nuget package, but it still felt very hacky and fragile.

Fortunately, the situation is about to get a whole lot better.

# .NET 7 to the rescue

.NET 7 is the next major release of .NET, and it's expected to bring something that has been in development for a long time: native AOT. The idea is to blur the line between native and managed applications, by precompiling the whole application and the runtime into native code. The main benefits are faster startup time, reduced memory consumption, possibly faster execution time in some situations, and the part that we're going to focus on today: easier interoperability with native code.

By using NativeAOT, we can fix most of the drawbacks mentioned earlier:

* Because the extension is bundled into a single file, we don't have to deal with dependencies

* Native exports are supported out of the box

* The runtime becomes part of the extension itself, so we can use any version of .NET (well, any version that supports NativeAOT)

So, how can we write WinDbg extensions with NativeAOT?

# Compiling using NativeAOT

First thing first, let's see how to compile an application with NativeAOT. Note that this part is probably going to change widely until the official release of .NET 7.

Create a class library project targeting .NET 6 then add a reference to the nuget package `Microsoft.DotNet.ILCompiler 7.0.0-*`. For that, you need to add `https://pkgs.dev.azure.com/dnceng/public/_packaging/dotnet-experimental/nuget/v3/index.json` to your nuget package sources (this can be done globally or with a `nuget.config` file next to your project). Then you need to publish your project with the command:

```bash
$ dotnet publish /p:NativeLib=Shared /p:SelfContained=true -r <RID> -c <Configuration>
```

So for instance:

```bash
$ dotnet publish /p:NativeLib=Shared /p:SelfContained=true -r win-x64 -c Release
```

And… that's it. This will produce a single DLL file embedding the runtime and compiled ahead of time. There are a few settings you can play with to tweak the output, but I won't cover them in this article. You can learn more about them [here](https://github.com/dotnet/runtimelab/tree/8e81d3a5bfd7639a197b51a1f65fcbba129d3b5f/samples/NativeLibrary) or [here](https://github.com/dotnet/runtimelab/blob/8e81d3a5bfd7639a197b51a1f65fcbba129d3b5f/docs/using-nativeaot/README.md).

# Writing a WinDbg extension

Now that we know how to use NativeAOT on our class library project, we still need to turn it into a valid WinDbg extension. The first step is to expose a [`DebugExtensionInitialize`](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/dbgeng/nc-dbgeng-pdebug_extension_initialize?WT.mc_id=DT-MVP-5003493) function, that will be called by WinDbg when loading the extension:

```csharp
[UnmanagedCallersOnly(EntryPoint = "DebugExtensionInitialize")]
public static unsafe int DebugExtensionInitialize(uint* version, uint* flags)
{
    *version = (1 & 0xffff) << 16;
    *flags = 0;
    return 0;
}
```

A few things to note here:

* The `UnmanagedCallersOnly` attribute instructs the compiler that the method is expected to be called from native code. When seeing this attribute, the NativeAOT compiler will automatically generate a native export (which is what we needed [UnmanagedExports](https://www.nuget.org/packages/UnmanagedExports/) for before).

* The last time I tried, I wasn't able to mix `ref`/`out` keywords with the `UnmanagedCallersOnly` attribute (which actually surprised me because I'm fairly sure it was working when I first tried a few months ago). That's why I'm using `uint*` pointers instead.

With that, the extension can be loaded into WinDbg, but we still need to add commands.

Every command is a function exported natively, [with the following signature](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/dbgeng/nc-dbgeng-pdebug_extension_call?WT.mc_id=DT-MVP-5003493):

```c++
HRESULT PdebugExtensionCall(
   [in]           PDEBUG_CLIENT Client,
   [in, optional] PCSTR Args 
)
```

Both parameters are pointers, so we can translate it in C# to:

```csharp
[UnmanagedCallersOnly(EntryPoint = "heapstat")]
public static int HeapStat(IntPtr client, IntPtr argsPtr)
{
    return 0;
}
```

Note that `MarshalAs` attributes are not currently supported, so we need to marshal the parameters ourselves. For instance for the `args` parameter it can be done like this:

```csharp
var args = Marshal.PtrToStringAnsi(argsPtr);
```

In theory we have everything we need to write a WinDbg extension. The instance of [`IDebugClient`](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/dbgeng/nn-dbgeng-idebugclient?WT.mc_id=DT-MVP-5003493) can be retrieved using the first parameter, and from there we have access to the full debugging API. But writing an extension using the raw debug API is a daunting task, and fortunately most of the heavy lifting can be done by ClrMD.

# Adding ClrMD to the mix

NativeAOT is not currently supported by ClrMD. It uses some unsupported functions, and some required interfaces are not exposed publicly. But it does not take a lot of effort to fix that, and [I've started experimenting with a fork](https://github.com/kevingosse/clrmd/tree/nativeaot) to make the necessary changes. I've also opened [an issue in the parent repository](https://github.com/microsoft/clrmd/issues/962) to probe for interest.

The first thing to do to use ClrMD in our extension is to create an instance of `DataTarget`. There is already an API for that, so this is fairly straightforward:

```csharp
DataTarget = DataTarget.CreateFromDbgEng(ptrClient);
```

At some point we will want to output some text to the debugger, and for that we're going to need the [`IDebugControl`](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/dbgeng/nn-dbgeng-idebugcontrol?WT.mc_id=DT-MVP-5003493) interface. All the boilerplate code needed to use this interface is already in ClrMD but not exposed publicly, so that's one of the changes that is implemented in my fork. I expose the debugger interfaces through the `IDbEng` interface, then it's just a matter of creating a custom stream that will call [`IDebugControl::ControlledOutput`](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/dbgeng/nf-dbgeng-idebugcontrol-controlledoutput?WT.mc_id=DT-MVP-5003493) whenever we write something to the console:

```csharp
var dbgEng = (IDbgEng)DataTarget.DataReader;

var stream = new StreamWriter(new DebugEngineStream(dbgEng.Control));
stream.AutoFlush = true;
Console.SetOut(stream);
```

The code of the `DebugEngineStream` can be found [in the ClrMD samples](https://github.com/kevingosse/clrmd/blob/nativeaot/src/Samples/NativeWindbgExtension/DebugEngineStream.cs).

Now we really have everything we need. We can create an instance of `ClrRuntime` from the `DataTarget` and write our code like in a classic ClrMD application:

```csharp
[UnmanagedCallersOnly(EntryPoint = "heapstat")]
public static int HeapStat(IntPtr client, IntPtr argsPtr)
{
    var args = Marshal.PtrToStringAnsi(argsPtr);

    // Initializes the instance of ClrRuntime and redirects the console output
    if (!InitApi(client))
    {
        return 0;
    }

    var heap = Runtime.Heap;

    var stats = from obj in heap.EnumerateObjects()
                group obj by obj.Type into g
                let size = g.Sum(p => (long)p.Size)
                orderby size
                select new
                {
                    Size = size,
                    Count = g.Count(),
                    g.Key.Name
                };

    foreach (var entry in stats)
    {
        Console.WriteLine("{0,12:n0} {1,12:n0} {2}", entry.Count, entry.Size, entry.Name);
    }

    return 0;
}
```

We can then publish the DLL with NativeAOT and load it into WinDbg like any extension:

{{<image classes="fancybox center" src="/images/writing-native-windbg-extensions-in-c-5390726f3cec-1.webp" >}}

The full code can be found [here](https://github.com/kevingosse/clrmd/tree/nativeaot/src/Samples/NativeWindbgExtension).
