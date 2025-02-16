---
url: writing-a-net-gc-in-c-part-4
title: Writing a .NET Garbage Collector in C#  -  Part 4
subtitle: Using NativeAOT to write a .NET GC in C#. In the fourth part, we see how walk the heap and how to keep track of allocation contexts.
summary: Using NativeAOT to write a .NET GC in C#. In the fourth part, we see how walk the heap and how to keep track of allocation contexts.
date: 2025-02-18
tags:
- dotnet
- nativeaot
- garbage-collection
author: Kevin Gosse
thumbnailImage: /images/2025-02-18-writing-a-net-gc-in-c-part-4-3.png
---

{{<rawhtml>}}
<a href="https://amzn.to/42tg58c" target="_blank" rel="noopener noreferrer" style="text-decoration: none; color: inherit;">
  <div style="display: flex; align-items: center; border: 2px solid #ccc; border-radius: 8px; padding: 10px; background-color: #f9f9f9; box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1); margin-top: 50px;">
    <span style="margin-right: 15px; font-size: 1.1em;">
      If you enjoy this article, make sure to check the 2nd edition of <strong><u>Pro .NET Memory Management</u></strong> for more insights on the .NET Garbage Collector internals!
    </span>
    <img src="/images/progc.png" alt="Pro .NET Memory Management" style="width: 100px; height: auto; border-radius: 4px;" />
  </div>
</a>
{{</rawhtml>}}

This is already the fourth part of our journey to write a .NET garbage collector in C#. This time, we will learn how to walk the managed heap, which is a prerequisite to implement the mark phase of the garbage collector. If you haven't read them or if you need a refresher, you can find the previous parts here:

- [Part 1:](https://minidump.net/2025-28-01-writing-a-net-gc-in-c-part-1/) Introduction and setting up the project
- [Part 2:](https://minidump.net/writing-a-net-gc-in-c-part-2/) Implementing a minimal GC
- [Part 3:](https://minidump.net/writing-a-net-gc-in-c-part-3/) Using the DAC to inspect the managed objects

# Walking the heap

So what is it about? We are implementing [a tracing garbage collector](https://en.wikipedia.org/wiki/Tracing_garbage_collection), which means that we will be following the chains of references to determine the reachability of the objects (as opposed to other forms of automatic memory management, like reference counting). Traversing the reference tree will tell us which objects are still in use, and by exclusion which ones are not. Those are the objects that we can safely collect. But to "exclude" objects, we need to know about all of them, and that's why we need to be able to walk the heap.

This is not the first time I walk the managed heap, and I warmly recommend you to read [this article](https://minidump.net/dumping-the-managed-heap-in-csharp/) as a complement to this one. 

The trick to walking the heap is to realize that the objects are forming a kind of linked list, at least as long as they're contiguous (more on this later). Starting from an object, we can find the next one by computing its size and adding it to the current address, and repeat that until we reach the end of the heap.

From a size perspective, there are two kind of objects: fixed-size objects and variable-size objects. The vast majority of objects are fixed-size, we can get their size simply by reading the `BaseSize` field of their method table. Arrays and strings have a variable length, so computing their size is a bit more complex. We must read their length, located right after the method table pointer, and multiply it by the size of each element, stored in the `ComponentSize` field of the method table. The result must be added to the base size to get the total size of the object.

With that information, we can already write a `ComputeSize` method, that takes a pointer to a managed object and returns its size. We will that method later in the article:

```csharp
    private static unsafe uint ComputeSize(GCObject* obj)
    {
        var methodTable = obj->MethodTable;

        if (!methodTable->HasComponentSize)
        {
            // Fixed-size object
            return methodTable->BaseSize;
        }

        // Variable-size object
        return methodTable->BaseSize + obj->Length * methodTable->ComponentSize;
    }
```

The `GCObject` struct was first defined [in part 2](https://minidump.net/writing-a-net-gc-in-c-part-2/):

```csharp
[StructLayout(LayoutKind.Sequential)]
public unsafe struct GCObject
{
    public MethodTable* MethodTable;
    public uint Length;
}
```

For the `MethodTable` struct, we can simply duplicate [the one used by some methods of the BCL](https://source.dot.net/#System.Private.CoreLib/src/System/Runtime/CompilerServices/RuntimeHelpers.CoreCLR.cs,3a48c25c1e3ec333), saving us quite some time.

We can now write a method that walks a portion of memory, again assuming that the objects are stored contiguously. We use [the `DacManager` from part 3](https://minidump.net/writing-a-net-gc-in-c-part-3/) to get the name of the objects:

```csharp
private void TraverseHeap(nint start, nint end)
{
    var ptr = start + IntPtr.Size;

    while (ptr < end)
    {
        var obj = (GCObject*)ptr;
        var name = _dacManager?.GetObjectName(new(ptr));

        Write($"{ptr:x2} - {name}");

        var alignment = sizeof(nint) - 1;
        ptr += ((nint)ComputeSize(obj) + alignment) & ~alignment;
    }
}
```

There are two things to note here. First, as mentioned multiple times in the past, references to managed objects don't point to the actual start of the object. We need to add `IntPtr.Size` to the start to skip the header and find the method table pointer. The size of the header is included in the base size of the object, so the next iterations of the loop will point to the right part of the object. Second, objects are aligned on a pointer boundary, so we must make sure to fix the alignment after each object.

Now that we know how to walk a range of memory, we still need to figure out all the ranges that may contain objects.

# Managing allocation contexts

For now, our GC has a very simple allocation strategy: every time a thread asks for a new allocation context, we allocate a new 32KB chunk of memory using `NativeMemory.AllocZeroed` (which effectively translates into a C `calloc` call). We need to keep track of them to be able to walk the heap. This can be done with a simple thread-safe list:

```csharp
private ConcurrentQueue<(IntPtr start, IntPtr end)> _allocationContexts = new();
```

In the `Alloc` method, we add the new allocation context to the list:

```csharp
    public GCObject* Alloc(ref gc_alloc_context acontext, nint size, GC_ALLOC_FLAGS flags)
    {
        var result = acontext.alloc_ptr;
        var advance = result + size;

        if (advance <= acontext.alloc_limit)
        {
            acontext.alloc_ptr = advance;
            return (GCObject*)result;
        }

        var growthSize = Math.Max(size, 32 * 1024) + IntPtr.Size;
        var newPages = (IntPtr)NativeMemory.AllocZeroed((nuint)growthSize);

        // NEW: we now keep track of the allocation context
        _allocationContexts.Enqueue((newPages, newPages + growthSize));

        var allocationStart = newPages + IntPtr.Size;
        acontext.alloc_ptr = allocationStart + size;
        acontext.alloc_limit = newPages + growthSize;

        return (GCObject*)allocationStart;
    }
```

We can now write a method to traverse the whole heap. It simply enumerates all the allocation contexts, and call the `TraverseHeap(nint start, nit end)` method that we wrote earlier:

```csharp
    private void TraverseHeap()
    {
        foreach (var (start, end) in _allocationContexts)
        {
            TraverseHeap(start, end);
        }
    }
```

We also need to make a small addition to the `TraverseHeap(nint start, nit end)` method: because the allocation context might not be full, we need to break the loop when we find a null method table pointer:

```csharp
private void TraverseHeap(nint start, nint end)
{
    var ptr = start + IntPtr.Size;

    while (ptr < end)
    {
        var obj = (GCObject*)ptr;

        if (obj->MethodTable == null)
        {
            // We reached the end of used part of the allocation context
            break;
        }

        var name = _dacManager?.GetObjectName(new(ptr));

        Write($"{ptr:x2} - {name}");

        var alignment = sizeof(nint) - 1;
        ptr += ((nint)ComputeSize(obj) + alignment) & ~alignment;
    }
}
```

The last step is to plug the heap traversal logic into the `GarbageCollect` method. We suspend the managed threads before walking the heap and resume them afterwards. This is not strictly necessary because we're not moving or free objects, but we definitely will at some point at the future:

```csharp
    public HResult GarbageCollect(int generation, bool low_memory_p, int mode)
    {
        Write("GarbageCollect");

        _gcToClr.SuspendEE(SUSPEND_REASON.SUSPEND_FOR_GC);

        _gcHandleManager.Store.DumpHandles(_dacManager);
        TraverseHeap();

        _gcToClr.RestartEE(finishedGC: true);

        return HResult.S_OK;
    }
```

If we try it in our test application, we can see the content of the heap as expected:

{{<image classes="fancybox center" src="/images/2025-02-18-writing-a-net-gc-in-c-part-4-1.png" >}}

This allocation strategy could certainly work in simple applications, and it would be interesting to benchmark it in the future and see how it compares to other solutions. Ultimately, the garbage collector has to compromise between multiple factors, and we've already introduced a few of them. The first one is the size of the allocation contexts. Each thread is given its own allocation context, if we give a bigger allocation context then it means that the thread is able to perform more allocations on its own, which reduces the CPU overhead. On the other hand, with smaller allocation contexts we reduce the memory usage of the application, by having a working set that more accurately represents how much memory is actually used.

The .NET GC has a strategy to balance both concerns: having small allocation contexts (8KB), while keeping the cost of allocating new contexts low. This is done by preallocating a bigger chunk of memory and assigning parts of it into allocation contexts as needed. Those chunks of memory are referred to as "segments".

# Switching to segments

To understand exactly how segments work, we're going to implement them in our GC. The first thing to do is to actually declare our segments. We use a simple class to keep track of their bounds:

```csharp
public unsafe class Segment
{
    public IntPtr Start;
    public IntPtr Current;
    public IntPtr End;

    public Segment(nint size)
    {
        Start = (IntPtr)NativeMemory.AllocZeroed((nuint)size);
        Current = Start;
        End = Start + size;
    }
}
```

We add a few fields to keep track of the segments, and we allocate the first one during initialization:

```csharp
    private const int AllocationContextSize = 32 * 1024;
    private const int SegmentSize = AllocationContextSize * 128;

    private List<Segment> _segments = new();
    private Segment _activeSegment;

    public HResult Initialize()
    {
        // ...

        _activeSegment = new(SegmentSize);
        _segments.Add(_activeSegment);
    }
```

One of the advantages of segments is that they eliminate the need to keep track of the allocation contexts, so we can remove the `_allocationContexts` collection that we added previously. Instead, we can directly walk the segments. There's a catch however. Our strategy to walk the heap was based on the assumption that objects are stored contiguously, without any gap between them. With allocation contexts pointing to a common segment, we break this invariant.

{{<image classes="fancybox center" src="/images/2025-02-18-writing-a-net-gc-in-c-part-4-2.png" >}}

One solution could be to update our heap walking logic to scan the gap byte by byte until we find the next object. That would be very inefficient. Instead, we're going to add a step at the beginning of the garbage collection to set the segment back to a walkable state. To do so, we will allocate a dummy variable-size object at the end of the used part of each allocation context, indicating the size of the gap. Our `TraverseHeap` method will see it as an ordinary object and simply jump over it.

{{<image classes="fancybox center" src="/images/2025-02-18-writing-a-net-gc-in-c-part-4-3.png" >}}

Now that we have a strategy, we need to update the `Alloc` method to use segments. The logic is going to get quite a bit more complex. There are 4 cases to consider:
- There is still enough room in the allocation context: here, nothing changes, we allocate the object as before and bump `alloc_ptr`.
- There is not enough room in the allocation context, but there is enough room in the current segment: we assign a new allocation context to the thread. If there is enough room left then we assign a fully-sized allocation context, otherwise we assign whatever space is left.
- There is not enough room in the allocation context, and there is not enough room in the current segment: we allocate a new segment then process with the previous case.
- The object is too big to fit in a segment: we allocate a new segment specifically for this object, and we give an empty allocation context to the thread (so that it asks for a new one next time).

On top of that, we must ensure that the allocation contexts have enough room to store the dummy object. The minimum size of an object in .NET, is 3 pointers, so 24 bytes in 64-bit mode. This is the same for the dummy object. Imagine we allocate a 128 bytes allocation context, and store a 120 bytes object in it: we would have no room left for the dummy object and we would end up with an 8 bytes gap that we can't fill. So instead, we make sure that the allocation context is at least 3 pointers bigger than the object we're trying to allocate, we reduce the `alloc_limit` to prevent the thread from using that extra space.

Factoring all this into the `Alloc` method, we get:

```csharp
    private static int SizeOfObject = sizeof(nint) * 3;

    public GCObject* Alloc(ref gc_alloc_context acontext, nint size, GC_ALLOC_FLAGS flags)
    {
        var result = acontext.alloc_ptr;
        var advance = result + size;

        if (advance <= acontext.alloc_limit)
        {
            // There is enough room left in the allocation context
            acontext.alloc_ptr = advance;
            return (GCObject*)result;
        }

        // We need to allocate a new allocation context
        var minimumSize = size + SizeOfObject;

        if (minimumSize > SegmentSize)
        {
            // We need a dedicated segment for this allocation
            var segment = new Segment(size);
            segment.Current = segment.End;

            lock (_largeSegments)
            {
                _largeSegments.Add(segment);
            }

            acontext.alloc_ptr = 0;
            acontext.alloc_limit = 0;

            return (GCObject*)(segment.Start + IntPtr.Size);
        }

        lock (_segments)
        {
            if (_activeSegment.Current + minimumSize >= _activeSegment.End)
            {
                // The active segment is full, allocate a new one
                _activeSegment = new Segment(SegmentSize);
                _segments.Add(_activeSegment);
            }

            var desiredSize = Math.Min(Math.Max(minimumSize, AllocationContextSize), _activeSegment.End - _activeSegment.Current);

            result = _activeSegment.Current + IntPtr.Size;
            _activeSegment.Current += desiredSize;

            acontext.alloc_ptr = result + size;
            acontext.alloc_limit = _activeSegment.Current - IntPtr.Size * 2;

            return (GCObject*)result;
        }
    }
```

Note that we decrease the `alloc_limit` by only `IntPtr.Size * 2`. We don't need 3 because the allocation is already shifted by `IntPtr.Size` [as explained in part 2](https://minidump.net/writing-a-net-gc-in-c-part-2/).

Also, note the use of locking to keep the state of the segments consistent. A good way to reduce the contention around the lock is to have multiple heaps, each with its own segments, and to affinitize each thread to a given heap. This is the strategy used by the .NET GC in server mode, and we will probably experiment with it in the future.

Leaving room for the dummy object is nice, but we still have to actually allocate it at some point. We want to do that whenever an allocation context is discarded, which can happen at two points:
- In the `Alloc` method, when we assign a new allocation context to the thread.
- When a thread dies, in which case it will call the `IGCHeap.FixAllocContext` method to give us a chance to clean up.

We implement the `FixAllocContext` method to allocate the dummy object, which will fill the unused space of the allocation context:

```csharp
    public unsafe void FixAllocContext(gc_alloc_context* acontext, void* arg, void* heap)
    {
        FixAllocContext(ref Unsafe.AsRef<gc_alloc_context>(acontext));
    }

    private void FixAllocContext(ref gc_alloc_context acontext)
    {
        if (acontext.alloc_ptr == 0)
        {
            return;
        }

        AllocateFreeObject(acontext.alloc_ptr, (uint)(acontext.alloc_limit - acontext.alloc_ptr));
        acontext = new();
    }

    private void AllocateFreeObject(nint address, uint length)
    {
        var freeObject = (GCObject*)address;
        freeObject->MethodTable = _freeObjectMethodTable;
        freeObject->Length = length;
    }
```

The method table of the dummy object is given to us by the execution engine. We retrieve it using `IGCToCLR.GetFreeObjectMethodTable` in the constructor:

```csharp

    private MethodTable* _freeObjectMethodTable;

    public GCHeap(IGCToCLRInvoker gcToClr)
    {
        _freeObjectMethodTable = (MethodTable*)gcToClr.GetFreeObjectMethodTable();
        
        // ...
    }
```



And we don't forget to call it from `Alloc`:

```csharp
    private static int SizeOfObject = sizeof(nint) * 3;

    public GCObject* Alloc(ref gc_alloc_context acontext, nint size, GC_ALLOC_FLAGS flags)
    {
        var result = acontext.alloc_ptr;
        var advance = result + size;

        if (advance <= acontext.alloc_limit)
        {
            // There is enough room left in the allocation context
            acontext.alloc_ptr = advance;
            return (GCObject*)result;
        }

        // We need to allocate a new allocation context
        FixAllocContext(ref acontext);

        var minimumSize = size + SizeOfObject;

        // ...
    }
```

We're almost there. You might have notice a problem: we use `FixAllocContext` to allocate the dummy object in the discarded allocation contexts, but what about those that are still in use at the time of the garbage collection? The trick is to use `IGCToCLR.GcEnumAllocContexts` to enumerate all the allocation contexts, and call `FixAllocContext` on each of them:

```csharp
    public HResult GarbageCollect(int generation, bool low_memory_p, int mode)
    {
        Write("GarbageCollect");

        _gcToClr.SuspendEE(SUSPEND_REASON.SUSPEND_FOR_GC);

        _gcHandleManager.Store.DumpHandles(_dacManager);

        var callback = (delegate* unmanaged<gc_alloc_context*, IntPtr, void>)&EnumAllocContextCallback;
        _gcToClr.GcEnumAllocContexts((IntPtr)callback, GCHandle.ToIntPtr(_handle));

        TraverseHeap();

        _gcToClr.RestartEE(finishedGC: true);

        return HResult.S_OK;
    }

    [UnmanagedCallersOnly]
    private static void EnumAllocContextCallback(gc_alloc_context* acontext, IntPtr arg)
    {
        var handle = GCHandle.FromIntPtr(arg);
        var gcHeap = (GCHeap)handle.Target!;
        gcHeap.FixAllocContext(ref Unsafe.AsRef<gc_alloc_context>(acontext));
    }
```

`GcEnumAllocContexts` expects a callback that will be called for each allocation context. Because it will be invoked from native code, the callback has to be static and decorated with `[UnmanagedCallersOnly]`. To retrieve the instance of the `GCHeap` class from the static method, we use a `GCHandle` that we pass as an argument to the callback. The `GCHandle` is created in the constructor and will be reused throughout the lifetime of the application.

```csharp
    private GCHandle _handle;

    public GCHeap(IGCToCLRInvoker gcToClr)
    {
        _handle = GCHandle.Alloc(this);
        
        // ...
    }
```

Last but not least, we update the `TraverseHeap` methods: `TraverseHeap()` now needs to enumerate the segments instead of the allocation contexts, and `TraverseHeap(nint start, nint end)` doesn't need to check for null anymore. In our debug code, we display "Free" when we encounter a dummy object (like you could see in WinDbg).

```csharp
    private void TraverseHeap()
    {
        foreach (var segment in _segments)
        {
            TraverseHeap(segment.Start, segment.Current);
        }
    }

    private void TraverseHeap(nint start, nint end)
    {
        var ptr = start + IntPtr.Size;

        while (ptr < end)
        {
            var obj = (GCObject*)ptr;

            var name = obj->MethodTable == _freeObjectMethodTable
                ? "Free"
                : _dacManager?.GetObjectName(new(ptr));

            Write($"{ptr:x2} - {name}");

            var alignment = sizeof(nint) - 1;
            ptr += ((nint)ComputeSize(obj) + alignment) & ~alignment;
        }
    }
```

If we run the application, we can see that the heap is now correctly traversed, with the dummy objects filling the gaps:
{{<image classes="fancybox center" src="/images/2025-02-18-writing-a-net-gc-in-c-part-4-4.png" >}}

# Conclusion

We now have a working garbage collector that uses segments to manage the allocation contexts. We know how to walk the heap and list all the objects, the next step will be to find which ones are still reachable.

The code of this article is available on [GitHub](https://github.com/kevingosse/ManagedDotnetGC/tree/Part4).

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
