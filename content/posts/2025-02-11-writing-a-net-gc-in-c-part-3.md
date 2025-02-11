---
url: writing-a-net-gc-in-c-part-3
title: Writing a .NET Garbage Collector in C#  -  Part 3
subtitle: Using NativeAOT to write a .NET GC in C#. The third part adds some tooling to inspect the objects stored on the heap.
summary: Using NativeAOT to write a .NET GC in C#. The third part adds some tooling to inspect the objects stored on the heap.
date: 2025-02-11
tags:
- dotnet
- nativeaot
- garbage-collection
author: Kevin Gosse
thumbnailImage: /images/2025-02-11-writing-a-net-gc-in-c-part-3-1.png
---

In the previous parts, we built the simplest possible GC that can run basic .NET applications. If you missed them, you can find them here:
- [Part 1:](https://minidump.net/2025-28-01-writing-a-net-gc-in-c-part-1/) Introduction and setting up the project
- [Part 2:](https://minidump.net/writing-a-net-gc-in-c-part-2/) Implementing a minimal GC

In the next articles, we will start looking into how to traverse the references tree to find the objects that are still reachable. But before getting there, it would be nice if we had a way to get more information about the objects stored on the heap, for debugging purposes.

To illustrate the problem, let's see the `GCHandleStore.DumpHandles` method that we implemented in the previous part:

```csharp
    public void DumpHandles()
    {
        Write("GCHandleStore DumpHandles");

        for (int i = 0; i < _handleCount; i++)
        {
            Write($"Handle {i} - {_store[i]}");
        }
    }
```

As a reminder, this is the implementation of our `ObjectHandle` struct and its `ToString` method:

```csharp
[StructLayout(LayoutKind.Sequential)]
public struct ObjectHandle
{
    public nint Object;
    public nint ExtraInfo;
    public HandleType Type;

    public override string ToString() => $"{Type} - {Object:x2} - {ExtraInfo:x2}";
}
```

In the test application, this prints:

```
[GC] GCHandleStore DumpHandles
[GC] Handle 0 - HNDTYPE_WEAK_SHORT - 00 - 00
[GC] Handle 1 - HNDTYPE_STRONG - 00 - 00
[GC] Handle 2 - HNDTYPE_WEAK_SHORT - 220881ae040 - 00
[GC] Handle 3 - HNDTYPE_STRONG - 220881ae040 - 00
[GC] Handle 4 - HNDTYPE_STRONG - 1dff1bc4d28 - 00
[GC] Handle 5 - HNDTYPE_STRONG - 1dff1bc6d20 - 00
[GC] Handle 6 - HNDTYPE_STRONG - 1dff1bc6d80 - 00
[GC] Handle 7 - HNDTYPE_STRONG - 1dff1bc6df8 - 00
[GC] Handle 8 - HNDTYPE_STRONG - 1dff1bc6e70 - 00
[GC] Handle 9 - HNDTYPE_PINNED - 1dff1bc6ee8 - 00
[GC] Handle 10 - HNDTYPE_WEAK_SHORT - 00 - 00
[GC] Handle 11 - HNDTYPE_STRONG - 00 - 00
[GC] Handle 12 - HNDTYPE_WEAK_SHORT - 1dff1bda7f8 - 00
[GC] Handle 13 - HNDTYPE_STRONG - 1dff1bda838 - 00
[GC] Handle 14 - HNDTYPE_WEAK_SHORT - 1dff1bda8a8 - 00
[GC] Handle 15 - HNDTYPE_STRONG - 1dff1bda8e8 - 00
[GC] Handle 16 - HNDTYPE_WEAK_SHORT - 1dff1bda588 - 00
[GC] Handle 17 - HNDTYPE_WEAK_SHORT - 220881b6218 - 00
[GC] Handle 18 - HNDTYPE_STRONG - 220881b6258 - 00
[GC] Handle 19 - HNDTYPE_WEAK_SHORT - 220881b62c8 - 00
[GC] Handle 20 - HNDTYPE_STRONG - 220881b6308 - 00
[GC] Handle 21 - HNDTYPE_WEAK_SHORT - 220881b5fc8 - 00
[GC] Handle 22 - HNDTYPE_WEAK_SHORT - 220881b63f0 - 00
[GC] Handle 23 - HNDTYPE_STRONG - 220881b6430 - 00
[GC] Handle 24 - HNDTYPE_WEAK_SHORT - 220881b65d8 - 00
[GC] Handle 25 - HNDTYPE_STRONG - 220881b6618 - 00
[GC] Handle 26 - HNDTYPE_WEAK_SHORT - 220881b7268 - 00
[GC] Handle 27 - HNDTYPE_STRONG - 220881b72a8 - 00
[GC] Handle 28 - HNDTYPE_WEAK_SHORT - 220881b7318 - 00
[GC] Handle 29 - HNDTYPE_STRONG - 220881b7358 - 00
[GC] Handle 30 - HNDTYPE_WEAK_SHORT - 220881b70f0 - 00
[GC] Handle 31 - HNDTYPE_DEPENDENT - 220881b7788 - 00
[GC] Handle 32 - HNDTYPE_WEAK_SHORT - 220881b6e90 - 00
```

That's a good start, but it would really help if we knew what type of objects `220881ae040` or `1dff1bc4d28` are pointing to. 

Normally, getting the type of an object in .NET is trivial: just call `GetType()` on it. In our handle store we only have the address of the object, but we can use some unsafe code to build a reference to it. Let's try it:

```csharp
    public void DumpHandles()
    {
        Write("GCHandleStore DumpHandles");

        for (int i = 0; i < _handleCount; i++)
        {
            var handle = _store[i];
            var output = $"Handle {i} - {_store[i]}";

            if (handle.Object != 0)
            {
                // Take the address of the pointer,
                // reinterpret it as a pointer to an object reference,
                // and dereference it to get the object reference.
                var obj = *(object*)&handle.Object;
                output += $" - Object type: {obj.GetType()}";
            }

            Write(output);
        }
    }
```

But if we test it, it immediately crashes at the first non-null object:

```
[GC] GCHandleStore DumpHandles
[GC] Handle 0 - HNDTYPE_WEAK_SHORT - 00 - 00
[GC] Handle 1 - HNDTYPE_STRONG - 00 - 00
Fatal error. Internal CLR error. (0x80131506)
   at System.GC.Collect()
   at Program.<Main>$(System.String[])
```

The problem here is that our GC is compiled with NativeAOT, which comes with its own runtime. This runtime is separate from the .NET runtime that the test application is using. The type systems are distinct and not interoperable, so we can't directly manipulate a .NET type in the NativeAOT runtime.

So... how can we get that type information? The GC has no API to get that information, because it's not something it would normally need. In theory we could manually parse the method table and the module metadata, and use that to find the name of the type. While that would make for an interesting article, we're going to look for easier solutions for now.

# The cooperative way

One way to get the type information is simply to ask the test application to provide it. The idea is to add a method to the test application that fetches the type of an object given its address, and the GC will call it when needed. It strongly couples the application to the GC, but it might be acceptable since we only need that information for debugging purposes.

The method will be called from NativeAOT so we decorate it with the `[UnmanagedCallersOnly]` attribute. It receives a buffer to write the type name to, and it returns the length of the string.

```csharp
    [UnmanagedCallersOnly]
    public static unsafe int GetType(IntPtr address, char* buffer, int capacity)
    {
        var destination = new Span<char>(buffer, capacity);

        var obj = *(object*)&address;
        var type = obj.GetType().ToString();
        var length = Math.Min(type.Length, capacity);
        type[..length].CopyTo(destination);

        return length;
    }
```

We also need to tell the GC the address of that method, so we add a p/invoke and call it at startup:

```csharp
    public static void Initialize()
    {
        SetGetTypeCallback(&GetType);
    }

    [DllImport("ManagedDotnetGC.dll")]
    private static extern void SetGetTypeCallback(delegate* unmanaged<IntPtr, char*, int, int> callback);

```

On the GC side, we export the `SetGetTypeCallback` method and store the argument in a field:

```csharp
    [UnmanagedCallersOnly(EntryPoint = "SetGetTypeCallback")]
    public static unsafe void SetGetTypeCallback(IntPtr callback)
    {
        GetTypeCallback = (delegate* unmanaged<IntPtr, char*, int, int>)callback;
    }

    internal static unsafe delegate* unmanaged<IntPtr, char*, int, int> GetTypeCallback;
```

Finally, we update `DumpHandles` to call that method:

```csharp
    public void DumpHandles()
    {
        Write("GCHandleStore DumpHandles");

        var buffer = new char[1000];

        fixed (char* p = buffer)
        {
            for (int i = 0; i < _handleCount; i++)
            {
                var handle = _store[i];
                var output = $"Handle {i} - {_store[i]}";

                if (handle.Object != 0)
                {
                    if (GetTypeCallback != null)
                    {
                        var size = GetTypeCallback(handle.Object, p, buffer.Length);
                        output += $" - Object type: {new string(buffer[..size])}";
                    }
                }

                Write(output);
            }
        }
    }
```

Unfortunately, it crashes when running it:

```a
[GC] GCHandleStore DumpHandles
[GC] Handle 0 - HNDTYPE_WEAK_SHORT - 00 - 00
[GC] Handle 1 - HNDTYPE_STRONG - 00 - 00
Fatal error. Invalid Program: attempted to call a UnmanagedCallersOnly method from managed code.
   at System.GC.Collect()
   at Program.<Main>$(System.String[])
```

As the message indicates, we're not allowed to call an `[UnmanagedCallersOnly]` method from managed code (same with `Marshal.GetDelegateForFunctionPointer`). However we're calling it from the GC, which is unmanaged code, so what's going on?

If we look for the error message in the .NET runtime source code, we find that it's thrown [from the method `ReversePInvokeBadTransition`](https://github.com/dotnet/runtime/blob/826d9313afff3c406df6f8c13a8d70bcbe4e34e8/src/coreclr/vm/dllimportcallback.cpp#L184-L196) (a "reverse p/invoke" is when native code calls into managed code). That method is called [from the entry-point of the reverse p/invoke](https://github.com/dotnet/runtime/blob/826d9313afff3c406df6f8c13a8d70bcbe4e34e8/src/coreclr/vm/dllimportcallback.cpp#L210-L212), when "preemptive mode" is disabled for the current thread:

```cpp
    // Verify the current thread isn't in COOP mode.
    if (pThread->PreemptiveGCDisabled())
        ReversePInvokeBadTransition();
```

I already explained what preemptive mode is [in my SuppressGCTransition article](https://minidump.net/suppressgctransition-b9a8a774edbd/), but here is a quick reminder: in .NET, threads run in either preemptive or cooperative mode. When the GC triggers a collection, it needs to make sure that no managed code is running. For that, it suspends all the threads that are in cooperative mode (or rather, it "cooperates" with them to suspend them at a safe spot). Threads in preemptive mode are not suspended, they're trusted to not run any managed code. So to summarize:
 - Managed code always runs in cooperative mode.
 - Native code usually runs in preemptive mode, but it can sometimes run in cooperative mode (when executing CLR functions that require accessing managed objects, or when a p/invoke is decorated with the `[SuppressGCTransition]` attribute).

The error message is telling us that the thread is in cooperative mode when calling the reverse p/invoke, which is not allowed.

{{< alert >}}
To be honest, I don't really understand **_why_** calling a reverse p/invoke from cooperative mode is not allowed. This is not a situation that would normally happen, so I assume they used that as a cheap way to make sure managed code doesn't mistakenly call an `[UnmanagedCallersOnly]` method. But this is just speculation.
{{< /alert >}}

But why is our thread in cooperative mode? The managed code is calling `GC.Collect`, which in turn calls the `GCInterface_Collect` function in the CLR through a `QCall`:

```csharp
        [LibraryImport(RuntimeHelpers.QCall, EntryPoint = "GCInterface_Collect")]
        private static partial void _Collect(int generation, int mode);
```

`QCall` is a special kind of p/invoke that is used to call into the CLR from managed code. Their exact behavior [is well documented](https://github.com/dotnet/runtime/blob/main/docs/design/coreclr/botr/corelib.md#calling-from-managed-to-native-code), and the documentation clearly states:

> QCall also switch to preemptive GC mode like a normal P/Invoke.

So our thread should really be in preemptive mode. Unless...

The answer lies [in the `GCInterface_Collect` method itself](https://github.com/dotnet/runtime/blob/826d9313afff3c406df6f8c13a8d70bcbe4e34e8/src/coreclr/vm/comutilnative.cpp#L858-L879):

```cpp
extern "C" void QCALLTYPE GCInterface_Collect(INT32 generation, INT32 mode)
{
    QCALL_CONTRACT;

    BEGIN_QCALL;

    //We've already checked this in GC.cs, so we'll just assert it here.
    _ASSERTE(generation >= -1);

    //We don't need to check the top end because the GC will take care of that.

    GCX_COOP();
    GCHeapUtilities::GetGCHeap()->GarbageCollect(generation, false, mode);

    END_QCALL;
}
```

The `GCX_COOP` macro is used to switch the thread to cooperative mode, right before the call to  `IGCHeap::GarbageCollect`. Since we're calling `DumpHandles` from the `GarbageCollect` method, this is why our reverse p/invoke is failing.

Is it a dead end? Not quite. The `IGCToClr` interface (that we receive from the `GC_Initialize` method, if you forgot about the previous parts) has dedicated methods to control the thread mode. We can use them to switch the thread to preemptive mode before calling the reverse p/invoke, and back to cooperative mode afterwards.

```csharp
    public void DumpHandles()
    {
        Write("GCHandleStore DumpHandles");

        bool isPreeptiveGCDisabled = _gcToClr.IsPreemptiveGCDisabled();

        if (isPreeptiveGCDisabled)
        {
            _gcToClr.EnablePreemptiveGC();
        }

        var buffer = new char[1000];

        fixed (char* p = buffer)
        {
            for (int i = 0; i < _handleCount; i++)
            {
                var handle = _store[i];
                var output = $"Handle {i} - {_store[i]}";

                if (handle.Object != 0)
                {
                    if (GetTypeCallback != null)
                    {
                        var size = GetTypeCallback(handle.Object, p, buffer.Length);
                        output += $" - Object type: {new string(buffer[..size])}";
                    }
                }

                Write(output);
            }
        }

        if (isPreeptiveGCDisabled)
        {
            _gcToClr.DisablePreemptiveGC();
        }
    }
```

Now if we run the test app again, we can finally see the type of the objects:

```
[GC] GCHandleStore DumpHandles
[GC] Handle 0 - HNDTYPE_WEAK_SHORT - 00 - 00
[GC] Handle 1 - HNDTYPE_STRONG - 00 - 00
[GC] Handle 2 - HNDTYPE_WEAK_SHORT - 21235ae1fa0 - 00 - Object type: System.Threading.Thread
[GC] Handle 3 - HNDTYPE_STRONG - 21235ae1fa0 - 00 - Object type: System.Threading.Thread
[GC] Handle 4 - HNDTYPE_STRONG - 1d19f5bbb48 - 00 - Object type: System.Object[]
[GC] Handle 5 - HNDTYPE_STRONG - 1d19f5bdb40 - 00 - Object type: System.Int32[]
[GC] Handle 6 - HNDTYPE_STRONG - 1d19f5bdba0 - 00 - Object type: System.OutOfMemoryException
[GC] Handle 7 - HNDTYPE_STRONG - 1d19f5bdc18 - 00 - Object type: System.StackOverflowException
[GC] Handle 8 - HNDTYPE_STRONG - 1d19f5bdc90 - 00 - Object type: System.ExecutionEngineException
[GC] Handle 9 - HNDTYPE_PINNED - 1d19f5bdd08 - 00 - Object type: System.Object
[GC] Handle 10 - HNDTYPE_WEAK_SHORT - 00 - 00
[GC] Handle 11 - HNDTYPE_STRONG - 00 - 00
[GC] Handle 12 - HNDTYPE_WEAK_SHORT - 1d19f5cb988 - 00 - Object type: System.Diagnostics.Tracing.EventSource+OverrideEventProvider
[GC] Handle 13 - HNDTYPE_STRONG - 1d19f5cb9c8 - 00 - Object type: System.Diagnostics.Tracing.EtwEventProvider
[GC] Handle 14 - HNDTYPE_WEAK_SHORT - 1d19f5cba38 - 00 - Object type: System.Diagnostics.Tracing.EventSource+OverrideEventProvider
[GC] Handle 15 - HNDTYPE_STRONG - 1d19f5cba78 - 00 - Object type: System.Diagnostics.Tracing.EventPipeEventProvider
[GC] Handle 16 - HNDTYPE_WEAK_SHORT - 1d19f5cb718 - 00 - Object type: System.Diagnostics.Tracing.NativeRuntimeEventSource
[GC] Handle 17 - HNDTYPE_WEAK_SHORT - 21235aea178 - 00 - Object type: System.Diagnostics.Tracing.EventSource+OverrideEventProvider
[GC] Handle 18 - HNDTYPE_STRONG - 21235aea1b8 - 00 - Object type: System.Diagnostics.Tracing.EtwEventProvider
[GC] Handle 19 - HNDTYPE_WEAK_SHORT - 21235aea228 - 00 - Object type: System.Diagnostics.Tracing.EventSource+OverrideEventProvider
[GC] Handle 20 - HNDTYPE_STRONG - 21235aea268 - 00 - Object type: System.Diagnostics.Tracing.EventPipeEventProvider
[GC] Handle 21 - HNDTYPE_WEAK_SHORT - 21235ae9f28 - 00 - Object type: System.Diagnostics.Tracing.RuntimeEventSource
[GC] Handle 22 - HNDTYPE_WEAK_SHORT - 21235aea350 - 00 - Object type: System.Diagnostics.Tracing.EventSource+OverrideEventProvider
[GC] Handle 23 - HNDTYPE_STRONG - 21235aea390 - 00 - Object type: System.Diagnostics.Tracing.EtwEventProvider
[GC] Handle 24 - HNDTYPE_WEAK_SHORT - 21235aea538 - 00 - Object type: System.Diagnostics.Tracing.EventSource+OverrideEventProvider
[GC] Handle 25 - HNDTYPE_STRONG - 21235aea578 - 00 - Object type: System.Diagnostics.Tracing.EventPipeEventProvider
[GC] Handle 26 - HNDTYPE_DEPENDENT - 21235aeae18 - 21235aeae48 - Object type: System.Object
[GC] Handle 27 - HNDTYPE_WEAK_SHORT - 21235aeb238 - 00 - Object type: System.Diagnostics.Tracing.EventSource+OverrideEventProvider
[GC] Handle 28 - HNDTYPE_STRONG - 21235aeb278 - 00 - Object type: System.Diagnostics.Tracing.EtwEventProvider
[GC] Handle 29 - HNDTYPE_WEAK_SHORT - 21235aeb2e8 - 00 - Object type: System.Diagnostics.Tracing.EventSource+OverrideEventProvider
[GC] Handle 30 - HNDTYPE_STRONG - 21235aeb328 - 00 - Object type: System.Diagnostics.Tracing.EventPipeEventProvider
[GC] Handle 31 - HNDTYPE_WEAK_SHORT - 21235aeb0c0 - 00 - Object type: System.Buffers.ArrayPoolEventSource
[GC] Handle 32 - HNDTYPE_DEPENDENT - 21235aec4b8 - 00 - Object type: System.Buffers.SharedArrayPoolThreadLocalArray[]
[GC] Handle 33 - HNDTYPE_WEAK_SHORT - 21235aeae60 - 00 - Object type: System.Buffers.SharedArrayPool`1[System.Char]
[GC] Handle 34 - HNDTYPE_WEAK_LONG - 21235b98528 - 00 - Object type: System.RuntimeType+RuntimeTypeCache
[GC] Handle 35 - HNDTYPE_WEAK_LONG - 21235b98668 - 00 - Object type: System.RuntimeType+RuntimeTypeCache
[GC] Handle 36 - HNDTYPE_WEAK_LONG - 21235b98740 - 00 - Object type: System.RuntimeType+RuntimeTypeCache
[GC] Handle 37 - HNDTYPE_WEAK_LONG - 21235b98818 - 00 - Object type: System.RuntimeType+RuntimeTypeCache
[GC] Handle 38 - HNDTYPE_WEAK_LONG - 21235b98908 - 00 - Object type: System.RuntimeType+RuntimeTypeCache
[GC] Handle 39 - HNDTYPE_WEAK_LONG - 21235b989f8 - 00 - Object type: System.RuntimeType+RuntimeTypeCache
[GC] Handle 40 - HNDTYPE_WEAK_LONG - 21235b98af0 - 00 - Object type: System.RuntimeType+RuntimeTypeCache
[GC] Handle 41 - HNDTYPE_WEAK_LONG - 21235b98bc0 - 00 - Object type: System.RuntimeType+RuntimeTypeCache
[GC] Handle 42 - HNDTYPE_WEAK_LONG - 21235b98cf0 - 00 - Object type: System.RuntimeType+RuntimeTypeCache
[GC] Handle 43 - HNDTYPE_WEAK_LONG - 21235b98e00 - 00 - Object type: System.RuntimeType+RuntimeTypeCache
[GC] Handle 44 - HNDTYPE_WEAK_LONG - 21235b98f18 - 00 - Object type: System.RuntimeType+RuntimeTypeCache
[GC] Handle 45 - HNDTYPE_WEAK_LONG - 21235b99038 - 00 - Object type: System.RuntimeType+RuntimeTypeCache
[GC] Handle 46 - HNDTYPE_WEAK_LONG - 21235b99148 - 00 - Object type: System.RuntimeType+RuntimeTypeCache
[GC] Handle 47 - HNDTYPE_WEAK_LONG - 21235b99248 - 00 - Object type: System.RuntimeType+RuntimeTypeCache
[GC] Handle 48 - HNDTYPE_WEAK_LONG - 21235b99360 - 00 - Object type: System.RuntimeType+RuntimeTypeCache
[GC] Handle 49 - HNDTYPE_WEAK_LONG - 21235b99470 - 00 - Object type: System.RuntimeType+RuntimeTypeCache
```

So we can expose a method from the test application and call it from the GC. However, it has one major shortcoming: when we finally implement an actual garbage collection, we will want to inspect objects while the heap is in an inconsistent state, which could lead to some unpredictable behavior. This was a fun exploration but we need to find a better way.

# The better way

To summarize, we need a way to inspect the managed objects without running any managed code. Well, this is exactly what debuggers do, so maybe we can use the same APIs that they use?

Those APIs are exposed in a standalone component bundled with the runtime, called the DAC. It encapsulates all the logic needed to interact with the data structures of the runtime.

The DAC is designed to be used with a variety of targets: live process, remote process, crash dump... To make that possible it abstracts basic operations, like reading and writing memory, into an `ICLRDataTarget` interface that must be implemented by the debugger. 
As usual, I converted the original C++ interface to C#, and used my [`NativeObjects` library](https://github.com/kevingosse/NativeObjects) to wrap it. For our simple use-case we only need to implement a few of the methods of the interface:
 - `ReadVirtual`: reads memory from the target
 - `GetMachineType`: gets the architecture of the target
 - `GetPointerSize`: gets the size of a pointer on the target
 - `GetImageBase`: gets the base address of a given module

 ```csharp
 public unsafe class ClrDataTarget : ICLRDataTarget, IDisposable
{
    private readonly NativeObjects.ICLRDataTarget _clrDataTarget;

    public ClrDataTarget()
    {
        _clrDataTarget = NativeObjects.ICLRDataTarget.Wrap(this);
    }

    public IntPtr ICLRDataTargetObject => _clrDataTarget;

    public HResult GetMachineType(out uint machine)
    {
        var architecture = RuntimeInformation.ProcessArchitecture;

        // https://learn.microsoft.com/en-us/windows/win32/sysinfo/image-file-machine-constants
        if (architecture == Architecture.X86)
        {
            machine = 0x14c; // IMAGE_FILE_MACHINE_I386
        }
        else if (architecture == Architecture.X64)
        {
            machine = 0x8664; // IMAGE_FILE_MACHINE_AMD64
        }
        else if (architecture == Architecture.Arm64)
        {
            machine = 0xaa64; // IMAGE_FILE_MACHINE_ARM64
        }
        else
        {
            machine = 0;
            return HResult.E_FAIL;
        }        

        return HResult.S_OK;
    }

    public HResult GetPointerSize(out uint size)
    {
        size = (uint)IntPtr.Size;
        return HResult.S_OK;
    }

    public HResult GetImageBase(char* moduleName, out CLRDATA_ADDRESS baseAddress)
    {
        var name = new string(moduleName);

        foreach (ProcessModule module in Process.GetCurrentProcess().Modules)
        {
            if (module.ModuleName == name)
            {
                baseAddress = new CLRDATA_ADDRESS(module.BaseAddress.ToInt64());
                return HResult.S_OK;
            }
        }

        baseAddress = default;
        return HResult.E_FAIL;
    }

    public HResult ReadVirtual(CLRDATA_ADDRESS address, byte* buffer, uint size, out uint done)
    {
        Unsafe.CopyBlock(buffer, (void*)(IntPtr)address.Value, size);
        done = size;
        return HResult.S_OK;
    }
}
 ```

 For all the other methods we simply return `E_NOTIMPL`. We also have to implement `IUnknown` but there's nothing special about it so I won't show it here.

 The `ReadVirtual` and `GetImageBase` methods use the `CLRDATA_ADDRESS` struct to represent addresses. Apparently this is a signed type [which causes conversion issues when accessing the top 2 GB of a 32-bit process](https://github.com/dotnet/runtime/blob/main/src/coreclr/debug/daccess/dacimpl.h#L31-L36). This is very confusing so I decided to just [steal the C# implementation that Lee Culver wrote for ClrMD](https://github.com/microsoft/clrmd/blob/fb8c39b99ed792d650640823ee022f9f16996fe2/src/Microsoft.Diagnostics.Runtime/DacInterface/ClrDataAddress.cs).

The next step is to load the DAC into the process. The DAC is stored in a shared library, stored in the same directory as the runtime. To locate the runtime, we look for the `coreclr.dll` module in the current process and extract the directory from its path. Because the name of the shared library depends on the platform, I added a short helper method to convert it.

```csharp
public class DacManager : IDisposable
{
    public static unsafe HResult TryLoad(out DacManager? dacManager)
    {
        var coreclr = GetLibraryName("coreclr");

        var module = Process.GetCurrentProcess().Modules
            .Cast<ProcessModule>()
            .FirstOrDefault(m => m.ModuleName == coreclr);

        if (module == null)
        {
            Log.Write($"{coreclr} not found");
            dacManager = null;
            return HResult.E_FAIL;
        }

        var dacPath = Path.Combine(
            Path.GetDirectoryName(module.FileName)!,
            GetLibraryName("mscordaccore"));

        if (!File.Exists(dacPath))
        {
            Log.Write($"The DAC wasn't found at the expected path ({dacPath})");
            dacManager = null;
            return HResult.E_FAIL;
        }

        var library = NativeLibrary.Load(dacPath);

        // TODO
    }

    private static string GetLibraryName(string name)
    {
        if (RuntimeInformation.IsOSPlatform(OSPlatform.Windows))
        {
            return $"{name}.dll";
        }
        
        if (RuntimeInformation.IsOSPlatform(OSPlatform.Linux))
        {
            return $"lib{name}.so";
        }
        
        if (RuntimeInformation.IsOSPlatform(OSPlatform.OSX))
        {
            return $"lib{name}.dylib";
        }
        
        throw new PlatformNotSupportedException();
    }
}
```

Once the DAC is loaded into the process, we need to call the `CLRDataCreateInstance` function and give it our `ICLRDataTarget` object. In return, it gives us an instance of `IUnknown`, on which we can call `QueryInterface` to retrieve an `ISOSDacInterface`, which exposes the features of the DAC.

```csharp
        var library = NativeLibrary.Load(dacPath);

        try
        {
            var export = NativeLibrary.GetExport(library, "CLRDataCreateInstance");
            var createInstance = (delegate* unmanaged[Stdcall]<in Guid, IntPtr, out IntPtr, HResult>)export;

            var dataTarget = new ClrDataTarget();
            var result = createInstance(IClrDataProcessGuid, dataTarget.ICLRDataTargetObject, out var pUnk);

            var unknown = NativeObjects.IUnknown.Wrap(pUnk);
            result = unknown.QueryInterface(ISOSDacInterface.Guid, out var sosDacInterfacePtr);

            dacManager = result ? new DacManager(library, sosDacInterfacePtr) : null;
            return result;
        }
        catch
        {
            NativeLibrary.Free(library);
            throw;
        }
```

Our `DacManager` class stores the reference to the `ISOSDacInterface` object in the `Dac` property. We can use it to implement a method that extracts the type of a managed object, given its address:

```csharp
    public unsafe string? GetObjectName(CLRDATA_ADDRESS address)
    {
        var result = Dac.GetObjectClassName(address, 0, null, out var needed);

        if (!result)
        {
            return null;
        }

        char* str = stackalloc char[(int)needed];
        result = Dac.GetObjectClassName(address, needed, str, out _);

        if (!result)
        {
            return null;
        }

        return new string(str);
    }
```

This is a classic Win32 pattern: we don't know in advance how big the name is going to be, so we first call the method with a null buffer to get the size, then we allocate a buffer of the right size and call the method again.

We can finally use the `DacManager` in our `DumpHandle` method, with the proper checks to keep the debugging API optional:

```csharp
    public void DumpHandles(DacManager? dacManager)
    {
        Write("GCHandleStore DumpHandles");

        for (int i = 0; i < _handleCount; i++)
        {
            ref var handle = ref _store[i];
            var output = $"Handle {i} - {handle}";

            if (dacManager != null && handle.Object != 0)
            {
                output += $" - {dacManager.GetObjectName(new(handle.Object))}";
            }

            Write(output);
        }
    }
```

If we run the test application again, we can see the types of the objects stored in the handle store. Note that all the `System.RuntimeType+RuntimeTypeCache` objects from the previous solution are gone, I assume they were allocated during the p/invoke or reverse p/invoke calls:

{{<image classes="fancybox center" src="/images/2025-02-11-writing-a-net-gc-in-c-part-3-1.png" >}}

# Conclusion

We took a small detour from implementing the GC, but it was a nice excuse to explore the GC modes and to learn how to use the DAC. Next time, we will finally start looking into how to implement the mark phase of our custom GC.

The code of this article is available on [GitHub](https://github.com/kevingosse/ManagedDotnetGC/tree/Part3).

{{<rawhtml>}}
<a href="https://amzn.to/42tg58c" target="_blank" rel="noopener noreferrer" style="text-decoration: none; color: inherit;">
  <div style="display: flex; align-items: center; border: 2px solid #ccc; border-radius: 8px; padding: 10px; background-color: #f9f9f9; box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1); margin-top: 50px;">
    <span style="margin-right: 15px; font-size: 1.1em;">
      Liked this article? Don't hesitate to check the 2nd edition of <strong><u>Pro .NET Memory Management</u></strong> for more insights on the .NET Garbage Collector internals!
    </span>
    <img src="/images/progc.png" alt="Pro .NET Memory Management" style="width: 100px; height: auto; border-radius: 4px;" />
  </div>
</a>
{{</rawhtml>}}
