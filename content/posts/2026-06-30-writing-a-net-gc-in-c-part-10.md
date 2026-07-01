---
url: writing-a-net-gc-in-c-part-10
title: 'Writing a .NET Garbage Collector in C#  -  Part 10: Finalizers'
subtitle: Using NativeAOT to write a .NET GC in C#. In this part, we add support for finalization.
summary: Using NativeAOT to write a .NET GC in C#. In this part, we add support for finalization.
date: 2026-06-30
tags:
- dotnet
- nativeaot
- garbage-collection
author: Kevin Gosse
thumbnailImage: /images/2026-06-30-writing-a-net-gc-in-c-part-10-1.png
---

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

Last time, we added support for frozen segments in our custom GC, and ended up completely revamping the allocation strategy to take advantage of the virtually unlimited address space in 64-bit processes. We also officially dropped support for 32-bit processes, for convenience. We're getting very close to a fully functional mark and sweep, but there's still a very important topic that we haven't covered: finalizers.

If you need a refresher, don't hesitate to read or re-read those past articles:

- [Part 1:](https://minidump.net/2025-28-01-writing-a-net-gc-in-c-part-1/) Introduction and setting up the project
- [Part 2:](https://minidump.net/writing-a-net-gc-in-c-part-2/) Implementing a minimal GC
- [Part 3:](https://minidump.net/writing-a-net-gc-in-c-part-3/) Using the DAC to inspect the managed objects
- [Part 4:](https://minidump.net/writing-a-net-gc-in-c-part-4/) Walking the managed heap
- [Part 5:](https://minidump.net/writing-a-net-gc-in-c-part-5/) Decoding the GCDesc to find the references of a managed object
- [Part 6:](https://minidump.net/writing-a-net-gc-in-c-part-6/) Implementing the mark and sweep phases
- [Part 7:](https://minidump.net/writing-a-net-gc-in-c-part-7/) Marking handles
- [Part 8:](https://minidump.net/writing-a-net-gc-in-c-part-8/) Interior pointers
- [Part 9:](https://minidump.net/writing-a-net-gc-in-c-part-9/) Frozen segments and new allocation strategy

If you have the level required to read those articles, I'm going to assume that you know what finalizers are. Still, just to make sure that we are on the same page, here are the bits that are going to matter:
 - Any object in .NET can implement a finalizer
 - Finalizer execution is non-deterministic: the runtime doesn't guarantee if and when the finalizers are going to run. It is however guaranteed that a finalizer won't run while the object is referenced
 - Finalizers run in a dedicated thread
 - The order of execution of finalizers is non-deterministic: if an object A references an object B and they both implement a finalizer, it is possible that B might be finalized before A
 - Critical finalizers are the one exception to this rule: they're guaranteed to run after "normal" finalizers
 - A finalizer is allowed to let a reference to the object escape the scope, therefore "resurrecting" the object. Resurrected objects can be re-registered for finalization.

 With that out of the way, let's start digging in.

 # Finalizers in the CLR

 The way finalizers are implemented in the CLR is, in my opinion, a bit surprising. Specifically, I feel like there is a problem of separation of concerns. I would have expected that the tracking of finalizable objects, and the actual finalization, were both handled by the execution engine (EE). The GC would just notify the EE whenever an object is no longer referenced. In practice, as we're going to see, almost everything is handled by the GC, and the EE is only involved to run the actual finalizer.

 In terms of API, the GC is required to implement the following methods:

 - `SetFinalizationRun`: handles the calls to `GC.SuppressFinalize` 
 - `RegisterForFinalization`: handles the calls to `GC.ReRegisterForFinalize`
 - `GetNextFinalizable`: called by the execution engine to get the next object that should be finalized
 - `GetExtraWorkForFinalization`: allows the GC to run its own work on the finalizer thread
 - `GetNumberOfFinalizable`: not used anymore as far as I can tell. It was used in .NET Framework to track the progress of the finalizer thread during shutdown, to abort finalization if it stopped making progress

 As we can see, the GC is expected to handle pretty much everything, and the execution engine just knocks on the door to get the next item to execute.

 So what's the plan? During garbage collection, we must know which objects are finalizable, so that we don't immediately reclaim their memory. One way to do so is to store them in a dedicated list. The original .NET GC calls this list "the finalization queue" (even though it's not a queue strictly speaking). At the end of the garbage collection, we need to tell the execution engine which objects must be finalized (= the ones that are finalizable and no longer referenced). But because finalization runs asynchronously and the execution engine only asks one object at a time (through `GetNextFinalizable`), we need to store those objects in a separate list. This one is called the f-reachable queue.

 # The finalization queue

 For the finalization queue, we declare a simple array with a count:

 ```csharp
private GCObject*[] _finalizationQueue = new GCObject*[16];
private int _finalizationQueueCount;
```

We're going to use it to keep track of finalizable objects. There are two entry points for finalizable objects. The first one is `RegisterForFinalization`, which is called when managed code calls `GC.ReRegisterForFinalize`:

```csharp
private readonly Lock _finalizationLock = new();

public bool RegisterForFinalization(int gen, GCObject* obj)
{
    Write($"Registering object at {(nint)obj:X} for finalization");

    lock (_finalizationLock)
    {
        if (_finalizationQueueCount >= _finalizationQueue.Length)
        {
            var newQueue = new GCObject*[_finalizationQueue.Length * 2];
            Array.Copy(_finalizationQueue, newQueue, _finalizationQueue.Length);
            _finalizationQueue = newQueue;
        }

        _finalizationQueue[_finalizationQueueCount++] = obj;
    }

    return true;
}
```

This part is fairly straightforward. We acquire the finalization lock, check that the array is big enough, and add the element. If the array is too small, we double its size. The `gen` argument indicates the generation of the object, but our GC isn't generational so we can just ignore it.

The other entry point is the `Alloc` function. Whenever a thread allocates a new object, if that object is finalizable, it will call `Alloc` with the proper flags. We just need to redirect the call to `RegisterForFinalization`:

```csharp
public GCObject* Alloc(ref gc_alloc_context acontext, nint size, GC_ALLOC_FLAGS flags)
{
    var obj = DoAlloc(ref acontext);

    if (flags.HasFlag(GC_ALLOC_FLAGS.GC_ALLOC_FINALIZE))
    {
        RegisterForFinalization(0, obj);
    }

    return obj;

    GCObject* DoAlloc(ref gc_alloc_context acontext)
    {
        // The old Alloc function
        // ...
    }
}
```

{{< alert >}}
If you remember from past articles, each thread is granted an allocation context, that it uses to allocate objects without calling the GC. They only call `Alloc` when their allocation context is full, at which time they're granted a new one. For finalizable objects however, the threads must call `Alloc` *for every allocation*. This makes the allocation of finalizable objects significantly more expensive than normal objects.
{{< /alert >}}

# The f-reachable queue

At the end of the mark phase, we must look at all the objects in the finalization queue that haven't been marked (which indicates that they're not referenced anymore). Those objects will be moved to the f-reachable queue, where they will wait for the execution engine to finalize them.

Because critical finalizers must run after all "normal" finalizers, I decided to create two distinct f-reachable queues:

```csharp
private readonly Queue<nint> _freachableQueue = new();
private readonly Queue<nint> _criticalFreachableQueue = new();
```

To populate them, we write a `PrepareForFinalization` function that will be called after marking the objects:

```csharp
private void PrepareForFinalization()
{
    int i = 0;

    while (i < _finalizationQueueCount)
    {
        var obj = _finalizationQueue[i];

        if (!obj->IsMarked())
        {
            if (obj->MethodTable->HasCriticalFinalizer)
            {
                _criticalFreachableQueue.Enqueue((nint)obj);
            }
            else
            {
                _freachableQueue.Enqueue((nint)obj);
            }

            _finalizationQueueCount--;
            _finalizationQueue[i] = _finalizationQueue[_finalizationQueueCount];
            _finalizationQueue[_finalizationQueueCount] = null;
        }
        else
        {
            i++;
        }
    }
}
```

We iterate over the finalization queue (no need for locking because it happens during garbage collection while the threads are suspended) and look for objects that aren't marked. When we find one, we add it to the proper f-reachable queue (critical finalizers have a special flag set in their method-table), and we remove it from the finalization queue. To remove it cheaply, we swap its position with the last element of the queue, then set it to null.

We can now cheaply implement `GetNumberOfFinalizable`:

```csharp
public nint GetNumberOfFinalizable() => _freachableQueue.Count + _criticalFreachableQueue.Count;
```

When the finalizer thread runs, it will call `GetNextFinalizable` to get the next object to finalize. We simply dequeue an item from the f-reachable queue, or if empty, the critical f-reachable queue:

```csharp
public GCObject* GetNextFinalizable()
{
    GCObject* obj = null;

    if (_freachableQueue.Count > 0)
    {
        obj = (GCObject*)_freachableQueue.Dequeue();
    }
    else if (_criticalFreachableQueue.Count > 0)
    {
        obj = (GCObject*)_criticalFreachableQueue.Dequeue();
    }

    return obj;
}
```

Finally, at the very end of the garbage collection, we wake up the finalizer thread:

```csharp
_gcToClr.RestartEE(finishedGC: true);
_gcToClr.EnableFinalization(GetNumberOfFinalizable() > 0);
```

`EnableFinalization` takes a single `foundFinalizers` argument. It indicates whether some objects are pending finalization. The finalizer thread might wake up even if the f-reachable queues are empty, because it also has other duties, such as cleaning unused syncblocks (more on this later) or freeing dead threads.

# Mark phase

It's time to look at what impact finalization has on the mark phase. We need to be careful because a number of factors can cause things to go subtly wrong. 

Our current mark phase looks like this:

```csharp
private void MarkPhase()
{
    ScanContext scanContext = default;
    scanContext.promotion = true;
    scanContext._unused1 = GCHandle.ToIntPtr(_handle);

    Write("Scan roots");
    var scanRootsCallback = (delegate* unmanaged<GCObject**, ScanContext*, uint, void>)&ScanRootsCallback;
    _gcToClr.GcScanRoots((IntPtr)scanRootsCallback, 2, 2, &scanContext);

    ScanHandles();
    ScanDependentHandles();
    ClearHandles();
}
```

We mark the roots reported by the execution engine (`GcScanRoots`), then the objects referenced by strong handles, then dependent handles, and finally we clear the weak and dependent handles that don't point to a valid object anymore. We now have a new source of roots: the f-reachable queues. Objects in the f-reachable queues are not referenced anymore, but we mustn't collect them before their finalizer has run. To handle this, we add a new `ScanForFinalization` method where we start by moving unmarked objects from the finalization queue to the f-reachable queues (the `PrepareForFinalization` that we wrote earlier), then we scan those new roots:

```csharp
private void ScanForFinalization()
{
    PrepareForFinalization();

    ScanContext scanContext = default;

    foreach (GCObject* obj in _freachableQueue)
    {
        ScanRoots(obj, &scanContext, default);
    }

    foreach (GCObject* obj in _criticalFreachableQueue)
    {
        ScanRoots(obj, &scanContext, default);
    }
}
```

Now, when should `ScanForFinalization` be called? One subtlety that we conveniently ignored in the previous articles is that there are two types of weak references: short and long. Short weak references become null during garbage collection when the object isn't reachable _by user code_ anymore, even if it's not finalized yet. Long weak references stay valid until the object is _completely_ out of reach, which means that it can survive finalization.

| Stage | Short weak ref | Long weak ref |
|---|---|---|
| Object allocated (reachable) | 🟢 | 🟢 |
| No more strong references (before GC) | 🟢 | 🟢 |
| Garbage collection (found unreachable) | 🔴 | 🟢 |
| Finalization (queued / finalizer running) | 🔴 | 🟢 |
| Resurrection (finalizer re-roots it) | 🔴 | 🟢 |
| Finalized, not resurrected (next GC) | 🔴 | 🔴 |

This picture isn't compatible with our simplistic mark phase. First, we must update `ClearHandles` to selectively clear a given type of handles, rather than all at once. Then, we must clear short weak handles *before* moving the objects from the finalization queue to the f-reachable queue (at which point they will be considered as rooted). This means, before calling `ScanForFinalization`:

```csharp
private void MarkPhase()
{
    ScanContext scanContext = default;
    scanContext.promotion = true;
    scanContext._unused1 = GCHandle.ToIntPtr(_handle);

    Write("Scan roots");
    var scanRootsCallback = (delegate* unmanaged<GCObject**, ScanContext*, uint, void>)&ScanRootsCallback;
    _gcToClr.GcScanRoots((IntPtr)scanRootsCallback, 2, 2, &scanContext);

    ScanHandles();
    ScanDependentHandles();
    ClearHandles([HandleType.HNDTYPE_WEAK_SHORT]);
    ScanForFinalization();
    // ... ?
}
```

But that's not all. Because `ScanForFinalization` potentially causes some objects to become rooted, we must run `ScanDependentHandles` once more, as it may cause some dependent handles to become valid again (if you don't understand why, this is explained in the [marking handles](https://minidump.net/writing-a-net-gc-in-c-part-7/) chapter). After that second dependent handles scan, we can finally clear the long weak handles and dependent handles that are still not pointing at a marked object.

```csharp
private void MarkPhase()
{
    ScanContext scanContext = default;
    scanContext.promotion = true;
    scanContext._unused1 = GCHandle.ToIntPtr(_handle);

    Write("Scan roots");
    var scanRootsCallback = (delegate* unmanaged<GCObject**, ScanContext*, uint, void>)&ScanRootsCallback;
    _gcToClr.GcScanRoots((IntPtr)scanRootsCallback, 2, 2, &scanContext);

    ScanHandles();
    ScanDependentHandles();
    ClearHandles([HandleType.HNDTYPE_WEAK_SHORT]);
    ScanForFinalization();
    ScanDependentHandles();
    ClearHandles([HandleType.HNDTYPE_WEAK_LONG, HandleType.HNDTYPE_DEPENDENT]);
}
```

# BIT_SBLK_FINALIZER_RUN

User code can influence the finalization status of an object at any time, using the `GC.SuppressFinalize` and `GC.ReRegisterForFinalize` methods. It would be prohibitively expensive for the GC to remove/re-add an object to the finalization queue every time they're called. To make this cheaper, one bit inside of each object header is reserved to track its finalization status: `BIT_SBLK_FINALIZER_RUN`. To access it easily, we add an `ObjectHeader` struct with a `HasFinalizerRun` property:

```csharp
[StructLayout(LayoutKind.Sequential)]
public struct ObjectHeader
{
    private const uint BIT_SBLK_FINALIZER_RUN = 0x40000000;

    private uint _value;

    public bool HasFinalizerRun
    {
        get => (_value & BIT_SBLK_FINALIZER_RUN) != 0;
        set
        {
            if (value)
            {
                _value |= BIT_SBLK_FINALIZER_RUN;
            }
            else
            {
                _value &= ~BIT_SBLK_FINALIZER_RUN;
            }
        }
    }
}
```

The header is exposed through our `GCObject` struct. As a reminder, the object header is located *before* the beginning of the object:

```csharp
[StructLayout(LayoutKind.Sequential)]
public unsafe ref struct GCObject
{
    public MethodTable* RawMethodTable;
    public uint Length;

    public ObjectHeader* Header
    {
        get
        {
            var ptr = (int*)Unsafe.AsPointer(ref this);
            return (ObjectHeader*)(ptr - 1);
        }
    } 

    // ...
}
```

{{< alert >}}
This code is only safe if your struct is guaranteed to never be boxed. You can enforce this by declaring your struct as `ref struct`, like in this example.
{{< /alert >}}


Once we have this helper, implementing `SetFinalizationRun` (which maps to `GC.SuppressFinalize`) becomes trivial:

```csharp
public void SetFinalizationRun(GCObject* obj)
{
    obj->Header->HasFinalizerRun = true;
}
```

In `RegisterForFinalization`, we must make sure to clear the flag, in case we have a finalized object being re-registered for finalization:

```csharp
public bool RegisterForFinalization(int gen, GCObject* obj)
{
    if (obj->Header->HasFinalizerRun)
    {
        obj->Header->HasFinalizerRun = false;
        return true;
    }

    lock (_finalizationLock)
    {
        if (_finalizationQueueCount >= _finalizationQueue.Length)
        {
            var newQueue = new GCObject*[_finalizationQueue.Length * 2];
            Array.Copy(_finalizationQueue, newQueue, _finalizationQueue.Length);
            _finalizationQueue = newQueue;
        }

        _finalizationQueue[_finalizationQueueCount++] = obj;
    }

    return true;
}
```

Finally, in `GetNextFinalizable`, we must make sure not to return an object which has the flag set:

```csharp
public GCObject* GetNextFinalizable()
{
    GCObject* obj = null;

    while (obj == null && (_criticalFreachableQueue.Count > 0 || _freachableQueue.Count > 0))
    {           
        if (_freachableQueue.Count > 0)
        {
            obj = (GCObject*)_freachableQueue.Dequeue();
        }
        else if (_criticalFreachableQueue.Count > 0)
        {
            obj = (GCObject*)_criticalFreachableQueue.Dequeue();
        }

        if (obj != null && obj->Header->HasFinalizerRun)
        {
            obj = null;
        }            
    }

    return obj;
}
```

# Leftovers

There are two additional mechanisms that we haven't covered. For the first one, I'm going to copy/paste [the official implementation of the finalizer of `WeakReference<T>` in .NET](https://source.dot.net/#System.Private.CoreLib/src/runtime/src/libraries/System.Private.CoreLib/src/System/WeakReference.T.cs,e9a953b23e026c76):

```csharp
~WeakReference()
{
    Debug.Fail(" WeakReference<T> finalizer should never run");
}
```

So `WeakReference<T>` is a finalizable type for which the finalizer "should never run"? And there is even an assertion to validate it? What is this witchcraft?

It turns out that .NET has a mechanism known as _eager finalization_. The idea is that, for a hardcoded set of types (which actually only contains `WeakReference` and `WeakReference<T>`), the finalizer is trivial and the GC is allowed (or maybe should I say _expected_ given the above assertion) to directly call it inline, without delegating to the finalizer thread. This is done through the `EagerFinalized` method exposed by the `IGCToClr` API. We must therefore make sure to call it before moving objects to the f-reachable queue:

```csharp
private void PrepareForFinalization()
{
    int i = 0;

    while (i < _finalizationQueueCount)
    {
        var obj = _finalizationQueue[i];

        if (!obj->IsMarked())
        {
            if (!_gcToClr.EagerFinalized(obj)) // If it returns true, then the object has been finalized inline
            {
                if (obj->MethodTable->HasCriticalFinalizer)
                {
                    _criticalFreachableQueue.Enqueue((nint)obj);
                }
                else
                {
                    _freachableQueue.Enqueue((nint)obj);
                }
            }

            _finalizationQueueCount--;
            _finalizationQueue[i] = _finalizationQueue[_finalizationQueueCount];
            _finalizationQueue[_finalizationQueueCount] = null;
        }
        else
        {
            i++;
        }
    }
}
```

That takes care of eager finalization. The last bit is the syncblock cache, though arguably not directly related to finalization. I am not going to give an entire course about syncblocks as this article is already quite long but in a nutshell: to lock on arbitrary objects, and to have a stable hashcode, the runtime needs to store some metadata for each object. The runtime tries as hard as possible to fit that metadata in the object header, but in many scenarios (for instance, a contended lock), the header isn't big enough. When that happens, the metadata is moved to a dedicated structure, named the syncblock, itself stored in a table named the syncblock cache. The bottom line is that _outside of the managed heap_ there is a table storing information about some of the objects of the heap. This causes a problem of separation of concerns: the syncblock cache is managed by the execution engine, but the lifetime of the objects is managed by the garbage collector. For the execution engine to be able to reclaim the syncblock when an object dies, it must be notified by the garbage collector somehow. This is done through the `SyncBlockCacheWeakPtrScan` API. It takes a callback, and that callback will be invoked for every object referenced by the syncblock cache. For each of those references, the GC must answer whether the object is alive or not. The easiest place to call it is at the end of the mark phase, when all reachable objects are marked and therefore reachability becomes a simple check:

```csharp
private void MarkPhase()
{
    ScanContext scanContext = default;
    scanContext.promotion = true;
    scanContext._unused1 = GCHandle.ToIntPtr(_handle);

    Write("Scan roots");
    var scanRootsCallback = (delegate* unmanaged<GCObject**, ScanContext*, uint, void>)&ScanRootsCallback;
    _gcToClr.GcScanRoots((IntPtr)scanRootsCallback, 2, 2, &scanContext);

    ScanHandles();
    ScanDependentHandles();
    ClearHandles([HandleType.HNDTYPE_WEAK_SHORT]);
    ScanForFinalization();
    ScanDependentHandles();
    ClearHandles([HandleType.HNDTYPE_WEAK_LONG, HandleType.HNDTYPE_DEPENDENT]);

    var weakPtrScanCallback = (delegate* unmanaged<GCObject**, nint, nint, nint, void>)&WeakPtrScanCallback;
    _gcToClr.SyncBlockCacheWeakPtrScan(weakPtrScanCallback, 0, 0);
}

[UnmanagedCallersOnly]
private static void WeakPtrScanCallback(GCObject** obj, nint extraInfo, nint lp1, nint lp2)
{
    if (!(*obj)->IsMarked())
    {
        *obj = null;
    }
}
```

# ~Part10()

This honestly got a lot more complex than what I was expecting when I started implementing finalization. Step by step, we are getting closer to a fully functional mark&sweep GC, though I purposefully left out some intricate topics such as unloadable assembly contexts. As usual, the full code is available on [GitHub](https://github.com/kevingosse/ManagedDotnetGC/).

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
