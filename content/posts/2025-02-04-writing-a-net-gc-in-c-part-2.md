---
url: writing-a-net-gc-in-c-part-2
title: Writing a .NET Garbage Collector in C#  -  Part 2
subtitle: Using NativeAOT to write a .NET GC in C#. The second part builds the simplest possible GC that can run basic .NET applications.
date: 2025-02-04
tags:
- dotnet
- nativeaot
- garbage-collection
author: Kevin Gosse
thumbnailImage: /images/2025-02-04-writing-a-net-gc-in-c-part-2-1.png
---

[In the first part](/posts/writing-a-net-gc-in-c-part-1), we did setup the project and fixed an initialization issue caused by the NativeAOT toolchain. In this second part, we're going to start the implementation of our GC. The target for now is to build the simplest possible GC that can run basic .NET applications. This GC will only allocate memory and never free it, similar to [Konrad Kokosa's bump-pointer GC](https://github.com/kkokosa/UpsilonGC/tree/master/src/ZeroGC.BumpPointer).

The first step is to write the native interfaces we're going to need. For now there are four of them:
 - `IGCToCLR`: exposes execution engine APIs that can be called by the GC (for instance to suspend the threads)
 - `IGCHeap`: the main GC API
 - `IGCHandleManager`: exposes APIs to create or destroy handles
 - `IGCHandleStore`: the underlying store used by the handle manager

`IGCToCLR` is provided by the runtime. The other three must be implemented by the GC.

To handle the interop with native code, we're going to use the same [`NativeObjects` library](https://github.com/kevingosse/NativeObjects) that I implemented [for the managed profiler](https://minidump.net/writing-a-net-profiler-in-c-part-5/). All we have to do is write the interface in C# and decorate it with the `NativeObject` attribute:

```csharp
[NativeObject]
public unsafe interface IGCHeap
{
    /// <summary>
    /// Returns whether or not the given size is a valid segment size.
    /// </summary>
    bool IsValidSegmentSize(nint size);

    /// <summary>
    /// Returns whether or not the given size is a valid gen 0 max size.
    /// </summary>
    bool IsValidGen0MaxSize(nint size);

    /// <summary>
    /// Gets a valid segment size.
    /// </summary>
    nint GetValidSegmentSize(bool large_seg = false);

    [...]
}
```

Then we can either get a native pointer to a managed implementation of the interface:

```csharp
var gcHeap = new GCHeap();
IntPtr gcHeapPtr = NativeObjects.IGCHeap.Wrap(gcHeap);
```

Or we can get a managed pointer to a native implementation of the interface:

```csharp
IntPtr ptr = ...
var gcHeap = NativeObjects.IGCHeap.Wrap(ptr);
```

The managed interfaces are just a conversion in C# [of the C++ interfaces exposed in .NET source code](https://github.com/dotnet/runtime/blob/main/src/coreclr/gc/gcinterface.h), so I won't detail them here.

# The handle store

GC Handles are a fundamental concept in the .NET runtime, so even our basic GC needs to have some level of support for them. Fortunately, because our GC never frees or moves memory, we can get away for now with a fairly straightforward implementation.

A GC handle is made of three information: the handle type (weak, strong, pinned, ...), the address of the object it points to, and a pointer-sized field for extra information. We can therefore represent them with this structure:

```csharp
[StructLayout(LayoutKind.Sequential)]
public struct ObjectHandle
{
    public nint Object;
    public nint ExtraInfo;
    public HandleType Type;

    public override string ToString() => $"{Type} - {Object:x2} - {ExtraInfo:x2}";
}

public enum HandleType
{
    HNDTYPE_WEAK_SHORT = 0,
    HNDTYPE_WEAK_LONG = 1,
    HNDTYPE_WEAK_DEFAULT = 1,
    HNDTYPE_STRONG = 2,
    HNDTYPE_DEFAULT = 2,
    HNDTYPE_PINNED = 3,
    HNDTYPE_VARIABLE = 4,
    HNDTYPE_REFCOUNTED = 5,
    HNDTYPE_DEPENDENT = 6,
    HNDTYPE_ASYNCPINNED = 7,
    HNDTYPE_SIZEDREF = 8,
    HNDTYPE_WEAK_NATIVE_COM = 9
}
```

Not all handle types have extra information so in theory we could save some space  by using specialized structures for each handle type, but we're not going to care for our simple GC.

For now we can store the handles in a simple array. We hardcode the maximum number of handles to 10.000. It's more than enough to run test apps, but because we never free memory, every long-running app will eventually run out of handles. That means we'll have to revisit this at some point.

```csharp
public unsafe class GCHandleStore : IGCHandleStore
{
    private const int MaxHandles = 10_000;

    private readonly NativeObjects.IGCHandleStore _nativeObject;
    private readonly ObjectHandle* _store;
    private int _handleCount;

    public GCHandleStore()
    {
        _nativeObject = NativeObjects.IGCHandleStore.Wrap(this);
        _store = (ObjectHandle*)NativeMemory.AllocZeroed(MaxHandles, (nuint)sizeof(ObjectHandle));
    }

    public IntPtr IGCHandleStoreObject => _nativeObject;

    // TODO: Implement IGCHandleStore methods
}
```

The `IGCHandleStoreObject` property exposes the native pointer that we will give to the runtime. We can now implement the creation of handles. There are multiple methods for that on the `IGCHandleStore` interface, but then don't require dedicated logic:

```csharp
    public unsafe ref ObjectHandle CreateHandleOfType(GCObject* obj, HandleType type)
    {
        return ref CreateHandleWithExtraInfo(obj, type, null);
    }

    public unsafe ref ObjectHandle CreateHandleOfType2(GCObject* obj, HandleType type, int heapToAffinitizeTo)
    {
        return ref CreateHandleWithExtraInfo(obj, type, null);
    }

    public unsafe ref ObjectHandle CreateDependentHandle(GCObject* primary, GCObject* secondary)
    {
        return ref CreateHandleWithExtraInfo(primary, HandleType.HNDTYPE_DEPENDENT, secondary);
    }

    public unsafe ref ObjectHandle CreateHandleWithExtraInfo(GCObject* obj, HandleType type, void* pExtraInfo)
    {
        var index = Interlocked.Increment(ref _handleCount) - 1;

        if (index >= MaxHandles)
        {
            Environment.FailFast("Too many handles");
        }

        ref var handle = ref _store[index];

        handle.Object = (nint)obj;
        handle.Type = type;
        handle.ExtraInfo = (nint)pExtraInfo;

        return ref handle;
    }
```

`GCObject*` represents a pointer to a managed object. Though we never dereference it for now, the `GCObject` structure mimics the layout of a managed object in the runtime. 

```csharp
[StructLayout(LayoutKind.Sequential)]
public readonly struct GCObject
{
    public readonly IntPtr MethodTable;
    public readonly int Length;
}
```

The `IGCHandleStore` interface also exposes a `ContainsHandle` method that doesn't seem to be used anywhere in the .NET runtime. We're still going to implement it because it's fairly straightforward. As a bonus, we also add a `DumpHandles` method that we can use to debug the GC:

```csharp
    public void DumpHandles()
    {
        Write("GCHandleStore DumpHandles");

        for (int i = 0; i < _handleCount; i++)
        {
            Write($"Handle {i} - {_store[i]}");
        }
    }

    public bool ContainsHandle(ref ObjectHandle handle)
    {
        var ptr = Unsafe.AsPointer(ref handle);
        return ptr >= _store && ptr < _store + _handleCount;
    }
```

And that's all we need for the moment. In the future we will probably move to a more sophisticated handle store, with a segment-based storage that can grow as needed, possibly with a free-list to reuse handles.

# The handle manager

It's honestly not clear to me why there are two different interfaces to manage the handles, `IGCHandleManager` and `IGCHandleStore` could have been easily merged. It seems to be a remnant of a time where there were multiple handle stores in the runtime (maybe something to do with appDomains?), but that's not the case anymore.
`IGCHandleManager` provides methods to access the underlying `IGCHandleStore`, as well as methods to read or set information on the handles (this way, the execution engine is not dependent of how the GC represents the handles in memory).

```csharp
internal unsafe class GCHandleManager : IGCHandleManager
{
    private readonly NativeObjects.IGCHandleManager _nativeObject;
    private readonly GCHandleStore _gcHandleStore;

    public GCHandleManager()
    {
        _gcHandleStore = new GCHandleStore();
        _nativeObject = NativeObjects.IGCHandleManager.Wrap(this);
    }

    public IntPtr IGCHandleManagerObject => _nativeObject;

    public GCHandleStore Store => _gcHandleStore;

    public bool Initialize()
    {
        return true;
    }

    public IntPtr GetGlobalHandleStore()
    {
        return _gcHandleStore.IGCHandleStoreObject;
    }

    public unsafe ref ObjectHandle CreateGlobalHandleOfType(GCObject* obj, HandleType type)
    {
        return ref _gcHandleStore.CreateHandleOfType(obj, type);
    }

    public ref ObjectHandle CreateDuplicateHandle(ref ObjectHandle handle)
    {
        ref var newHandle = ref _gcHandleStore.CreateHandleOfType((GCObject*)handle.Object, handle.Type);
        newHandle.ExtraInfo = handle.ExtraInfo;
        return ref newHandle;
    }

    public unsafe void SetExtraInfoForHandle(ref ObjectHandle handle, HandleType type, nint extraInfo)
    {
        handle.ExtraInfo = extraInfo;
    }

    public unsafe nint GetExtraInfoFromHandle(ref ObjectHandle handle)
    {
        return handle.ExtraInfo;
    }

    public unsafe void StoreObjectInHandle(ref ObjectHandle handle, GCObject* obj)
    {
        handle.Object = (nint)obj;
    }

    public unsafe bool StoreObjectInHandleIfNull(ref ObjectHandle handle, GCObject* obj)
    {
        var result = InterlockedCompareExchangeObjectInHandle(ref handle, obj, null);        
        return result == null;
    }

    public unsafe void SetDependentHandleSecondary(ref ObjectHandle handle, GCObject* obj)
    {
        handle.ExtraInfo = (nint)obj;
    }

    public unsafe GCObject* GetDependentHandleSecondary(ref ObjectHandle handle)
    {
        return (GCObject*)handle.ExtraInfo;
    }

    public unsafe GCObject* InterlockedCompareExchangeObjectInHandle(ref ObjectHandle handle, GCObject* obj, GCObject* comparandObject)
    {
        return (GCObject*)Interlocked.CompareExchange(ref handle.Object, (nint)obj, (nint)comparandObject);
    }

    public HandleType HandleFetchType(ref ObjectHandle handle)
    {
        return handle.Type;
    }
}
```

Note: I'm only showing the methods that I implemented. There are a bunch of other methods, mostly related to freeing handles, that we don't need for now.

# IGCHeap

`IGCHeap` is the meat of the GC. It's really what you would have in mind when hearing "GC API". It's a pretty big interface, with a hefty 88 methods. Fortunately, we only need to implement a few of them for our basic GC.

The constructor of our `GCHeap` class is very similar to what we did for the handle store and handle manager, preparing the wrappers that we will need for native interop:

```csharp
internal unsafe class GCHeap : Interfaces.IGCHeap
{
    private readonly IGCToCLRInvoker _gcToClr;
    private readonly GCHandleManager _gcHandleManager;

    private readonly IGCHeap _nativeObject;

    public GCHeap(IGCToCLRInvoker gcToClr)
    {
        _gcToClr = gcToClr;
        _gcHandleManager = new GCHandleManager();

        _nativeObject = IGCHeap.Wrap(this);
    }
}

    public IntPtr IGCHeapObject => _nativeObject;
    public IntPtr IGCHandleManagerObject => _gcHandleManager.IGCHandleManagerObject;
```

The first method to implement is `Initialize`. As you can guess, this is called early during the runtime initialization, to give a chance to the GC to prepare everything it's going to need when managed code runs. The real .NET GC uses it to compute the size of the segments or regions, preallocate the heap, and so on. We don't need to do any of that, however we need to setup the write barrier.

Without going too much into the details, the write barrier is a code that is executed every time a reference is written to a field of an object. It updates a data structure known as the card table, which is used by the GC to track which references have been updated since the last garbage collection. This is an example of how the standalone GC API is really an abstraction built around the standard .NET GC, rather than a well-thought API for building your own GC. Even though we're not going to use the card table, there is no supported way to disable or modify the write barrier, so we have to setup everything properly.

Fortunately there is a trick, used by Konrad Kokosa in his own GC implementation. When running in workstation GC, the write barrier checks if the target address is within the GC range before writing to the card table. We can therefore set that range in such a way that all addresses will be outside of it. This effectively disables the write barrier.

```csharp
    public HResult Initialize()
    {
        Write("Initialize GCHeap");

        var parameters = new WriteBarrierParameters
        {
            operation = WriteBarrierOp.Initialize,
            is_runtime_suspended = true,
            ephemeral_low = -1 // nuint.MaxValue
        };

        _gcToClr.StompWriteBarrier(&parameters);

        return HResult.S_OK;
    }
```

By setting the lowest address of the GC range (`ephemeral_low`) to the maximum value, we ensure that all addresses are outside of the range. Without that trick, we would have to allocate memory for the card table, and assign it to the `card_table` field of the `WriteBarrierParameters` structure. The reason I prefer not doing it at this point is that the card table must be big enough to cover the entire heap, which requires either some bookkeeping to keep track of the bounds of the heap, or to preallocate a huge card table (each byte of the card table covers 2KB of memory).

Unfortunately, when using server GC, the write barrier doesn't check the range so this trick won't work. For now, we will just consider that server GC isn't supported (what does "server GC" even means for us? This is another example of how the .NET GC concepts leak through the standalone GC API).

The next method to implement is `Alloc`. This is arguably the most important method of the GC, called whenever a thread needs memory to allocate a new object. To implement it properly, we must introduce the concept of "allocation context". I already briefly described it [in a previous article](https://minidump.net/memory-alignment-of-doubles-in-c-1d13e3ce741/). The idea is that it would be very inefficient if every thread had to ask the GC for memory every time it needs to allocate an object. Instead, the GC gives each thread a chunk of memory, called the "allocation context", that it can use to allocate objects on its own. The thread will only call the GC when the allocation context is full, or in a number of special cases (such as the allocation of a finalizable object).

This is what the allocation context looks like:

```csharp
[StructLayout(LayoutKind.Sequential)]
public unsafe struct gc_alloc_context
{
    public nint alloc_ptr;
    public nint alloc_limit;
    public long alloc_bytes;
    public long alloc_bytes_uoh;

    public void* gc_reserved_1;
    public void* gc_reserved_2;
    public int alloc_count;
}
```

`alloc_ptr` is the current position in the allocation context, and `alloc_limit` is the end of the allocation context. In addition, there are a number of fields tracking stats about the allocations performed by that thread, as well as two pointer-sized fields that are used at the discretion of the GC (the standard GC uses them, for instance, to track which heap the thread is affinitized to, I took advantage of this [in a previous article](https://minidump.net/dumping-the-managed-heap-in-csharp/)).

For now, we will keep our allocation strategy simple: when `Alloc` is called, we first check if there is room left in the allocation context, in which case we simply bump `alloc_ptr`. If the allocation context is not big enough, we allocate a chunk of 32KB and use it as the new allocation context. We must also handle the special case when an object bigger than 32KB is allocated, in which case we return an allocation context of the exact size needed.

```csharp
    public GCObject* Alloc(ref gc_alloc_context acontext, nint size, GC_ALLOC_FLAGS flags)
    {
        var result = acontext.alloc_ptr;
        var advance = result + size;

        if (advance <= acontext.alloc_limit)
        {
            // The allocation context is big enough for this allocation
            acontext.alloc_ptr = advance;
            return (GCObject*)result;
        }

        // The allocation context is too small, we need to allocate a new one
        var growthSize = Math.Max(size, 32 * 1024) + IntPtr.Size;
        var newPages = (IntPtr)NativeMemory.AllocZeroed((nuint)growthSize);

        var allocationStart = newPages + IntPtr.Size;
        acontext.alloc_ptr = allocationStart + size;
        acontext.alloc_limit = newPages + growthSize;

        return (GCObject*)allocationStart;
    }
```

You might notice that we shift the allocation context by `IntPtr.Size` bytes. This is because references to managed objects don't point to the beginning of the object. Instead, they point to the method table pointer of the object, and there is a pointer-sized field before it to store the object header.

The last method of `IGCHeap` that we're going to implement is `GarbageCollect`. We will not actually do a garbage collection (yet), but we're going to use it to display some debug information (for now, just dumping the handles).
```csharp
    public HResult GarbageCollect(int generation, bool low_memory_p, int mode)
    {
        Write("GarbageCollect");
        _gcHandleManager.Store.DumpHandles();
        return HResult.S_OK;
    }
```

# Almost there

We have implemented all the required interfaces, we can now wire them up in the `GC_Initialize` method.

```csharp
    [UnmanagedCallersOnly(EntryPoint = "_GC_Initialize")]
    public static unsafe HResult GC_Initialize(IntPtr clrToGC, IntPtr* gcHeap, IntPtr* gcHandleManager, GcDacVars* gcDacVars)
    {
        Write("GC_Initialize");

        var clrToGc = NativeObjects.IGCToCLR.Wrap(clrToGC);
        var gc = new GCHeap(clrToGc);

        *gcHeap = gc.IGCHeapObject;
        *gcHandleManager = gc.IGCHandleManagerObject;

        return HResult.S_OK;
    }
```

Because our GC is not compatible with server GC, we're going to add a bit of code to fetch the configuration using the `IClrToGC` interface, and fail if server GC is enabled. The native `IGCToCLR.GetBooleanConfigValue` method expects strings that are encoded in one byte per character, whereas the .NET string are two bytes per character (UTF-16). Instead of converting them, we use the `u8` suffix to directly get an UTF-8 string.

```csharp
    [UnmanagedCallersOnly(EntryPoint = "_GC_Initialize")]
    public static unsafe HResult GC_Initialize(IntPtr clrToGC, IntPtr* gcHeap, IntPtr* gcHandleManager, GcDacVars* gcDacVars)
    {
        Write("GC_Initialize");

        var clrToGc = NativeObjects.IGCToCLR.Wrap(clrToGC);

        fixed (byte* privateKey = "gcServer"u8, publicKey = "System.GC.Server"u8)
        {
            clrToGc.GetBooleanConfigValue(privateKey, publicKey, out var gcServerEnabled);

            if (gcServerEnabled)
            {
                Write("This GC isn't compatible with server GC. Set DOTNET_gcServer=0 to disable it.");
                return HResult.E_FAIL;
            }
        }

        var gc = new GCHeap(clrToGc);

        *gcHeap = gc.IGCHeapObject;
        *gcHandleManager = gc.IGCHandleManagerObject;

        return HResult.S_OK;
    }
```

To test, I built a simple console application that allocates a few objects (including a large one), manipulates a `DependentHandle` (to validate the `IHandleStore` code), and finishes with a call to `GC.Collect()` (to trigger the `DumpHandles` code). Nothing impressive but it runs to completion without crashing. Most of the messages in the console are to identify GC methods that are called by the runtime but not yet implemented.

{{<image classes="fancybox center" src="/images/2025-02-04-writing-a-net-gc-in-c-part-2-1.png" >}}

As a more serious test, I also tried running the dashboard application in [OrchardCore.Samples](https://github.com/OrchardCMS/OrchardCore.Samples) (after upgrading it to .NET 9 and disabling server GC) and likewise I couldn't spot any obvious error.

{{<image classes="fancybox center" src="/images/2025-02-04-writing-a-net-gc-in-c-part-2-2.png" >}}

Of course, we're leaking all the allocated memory and there is a hard limit on the total number of handles created, so the app is bound to crash after a while. There is still a lot of work left to build a GC that can run real-world applications.

The code of this article is available on [GitHub](https://github.com/kevingosse/ManagedDotnetGC/tree/Part2).

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
