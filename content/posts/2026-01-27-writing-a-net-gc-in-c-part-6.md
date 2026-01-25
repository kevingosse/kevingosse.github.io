---
url: writing-a-net-gc-in-c-part-6
title: Writing a .NET Garbage Collector in C#  -  Part 6
subtitle: Using NativeAOT to write a .NET GC in C#. In the sixth part, we start implementing the mark phase of the garbage collection.
summary: Using NativeAOT to write a .NET GC in C#. In the sixth part, we start implementing the mark phase of the garbage collection.
date: 2026-01-27
tags:
- dotnet
- nativeaot
- garbage-collection
author: Kevin Gosse
thumbnailImage: /images/2026-01-27-writing-a-net-gc-in-c-part-6-1.jpg
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

After a long (way too long) break, it's time to resume our journey towards building a .NET garbage collector in C#. In the previous parts, we saw how to implement the minimal set of GC APIs to allow a simple application to run, and how to lay out the objects in memory to make the heap walkable. We then learned how to find the references of a given managed object. If you need a refresher, don't hesitate to jump back to those past articles:

- [Part 1:](https://minidump.net/2025-28-01-writing-a-net-gc-in-c-part-1/) Introduction and setting up the project
- [Part 2:](https://minidump.net/writing-a-net-gc-in-c-part-2/) Implementing a minimal GC
- [Part 3:](https://minidump.net/writing-a-net-gc-in-c-part-3/) Using the DAC to inspect the managed objects
- [Part 4:](https://minidump.net/writing-a-net-gc-in-c-part-4/) Walking the managed heap
- [Part 5:](https://minidump.net/writing-a-net-gc-in-c-part-5/) Decoding the GCDesc to find the references of a managed object

Now we have all the pieces of the puzzle to start implementing the mark phase of our garbage collection. The goal of the mark phase is to find all the objects that are currently reachable by user code, to deduce which ones aren't reachable anymore and can be freed.

# Marking

Marking starts from the roots. That is: references that the GC treats as unconditionally live at the beginning of a collection. The roots can be sorted into three buckets:  the local variables and thread-static storage, the GC handles, and the finalization queue. You might also think of static fields, however in practice the static variables are kept alive by GC handles.

While the GC is responsible for the last two buckets, the first one is handled directly by the runtime. The `IGCToCLR` interface exposes a `GcScanRoots` method that takes a callback. The callback will be called for every local variable. In addition to the callback, `GcScanRoots` method takes 3 arguments: `condemned` and `max_gen` which are only used for some corner case with server GC and so are largely inconsequential for us, and a `ScanContext`:

```csharp
[StructLayout(LayoutKind.Sequential)]
public struct ScanContext
{
    public IntPtr thread_under_crawl;
    public int thread_number;
    public int thread_count;
    public IntPtr stack_limit; // Lowest point on the thread stack that the scanning logic is permitted to read
    public bool promotion; //TRUE: Promotion, FALSE: Relocation.
    public bool concurrent; //TRUE: concurrent scanning
    public IntPtr _unused1;
    public IntPtr pMD;
    public int _unused3;
}
```

Likewise, most of the fields in `ScanContext` are only useful for server GC (where threads are affinitized to a given heap). We're only going to use two of them:
 - `promotion` tells the execution engine whether we're scanning for the promotion (where we're going to mark objects that are going to be promoted to an upper generation) or for the relocation (where we're going to update pointers after moving objects in memory). As far as I can tell, it mostly impacts what roots are reported when _conservative mode_ is enabled, so it shouldn't impact us for now. Still, we're going to set it to `true`.
 - `_unused1` is a pointer-sized field that the GC can fill to its own discretion. We're going to use it to store a pointer to our instance of `GCHeap`. We need it because the callback given to `GcScanRoots` must be decorated with `UnmanagedCallersOnly` because it will be called from native code, and methods decorated with this attribute must be static.

 ```csharp
private void MarkPhase()
{
    ScanContext scanContext = default;
    scanContext.promotion = true;
    scanContext._unused1 = GCHandle.ToIntPtr(_handle);

    var scanRootsCallback = (delegate* unmanaged<GCObject**, ScanContext*, uint, void>)&ScanRootsCallback;
    _gcToClr.GcScanRoots((IntPtr)scanRootsCallback, 2, 2, &scanContext);
}

[UnmanagedCallersOnly]
private static void ScanRootsCallback(GCObject** obj, ScanContext* context, uint flags)
{
    var handle = GCHandle.FromIntPtr(context->_unused1);
    var gcHeap = (GCHeap)handle.Target!;
    gcHeap.ScanRoots(*obj, context, (GcCallFlags)flags);
}

private void ScanRoots(GCObject* obj, ScanContext* context, GcCallFlags flags)
{
    // TODO: actual implementation of the callback
}
 ```

{{< alert >}}
Don't worry, I wasn't planning on dropping the term 'conservative mode' without explaining it. Conservative mode is enabled by setting `DOTNET_gcConservative=1`. It switches the execution engine from "precise" root tracking (where .NET knows exactly what is a root and what isn't) to "conservative" root tracking. In conservative root tracking, the execution engine scans the whole stack and reports any value that points to the range of memory managed by the GC. It greatly complicates the work of the GC because any reported root could be a false positive. As I understand, it's mostly meant for new environments where the CLR isn't fully implemented and doesn't support precise root tracking yet. It can also be used to test some new features. At this point, I have no plan to implement support for conservative mode in our custom GC.
{{< /alert >}}

Inside of our `ScanRoots` callback, we need to discover all the outgoing references from the given object, then browse the reference tree and mark all the objects that we discover. We've already seen [in part 5](https://minidump.net/writing-a-net-gc-in-c-part-5/) how to get the reference from a given object, and we implemented it into an `EnumerateObjectReferences` method. To mark the objects, we need to store somewhere the information that we found a given object. There are a few ways to do that. For instance, on x64 the object header includes 4 bytes of padding to keep the alignment, and we could theoretically repurpose that space. However, I plan to use them later to implement different kind of optimizations for other issues that the GC is going to face. Instead, we're going to do the same thing as the actual .NET GC. As a reminder, the layout of an object in memory is this:

```b                                     
                      Object         
                +-----------------+  
                | Object header   |  
                +-----------------+  
Object ref ---> | MethodTable*    | 
                +-----------------+ 
                |  Field1         | 
                +-----------------+ 
                |  Field2         | 
                +-----------------+ 
                |  Field3         |
                +-----------------|
                |  Field4         |
                +-----------------+
```

The trick is to take advantage of the fact that the method-table is aligned on a pointer boundary. It means that the two least-significant bits of the method-table pointer are always going to be 0 on a 32-bit runtime (and 3 bits on a 64-bit runtime). Whenever it marks an object, the GC just sets the least-significant bit of the method-table pointer to 1. Of course, it must make sure to restore the original pointer at the end of the garbage collection, before resuming the runtime.

Accordingly, we add the `Mark` and `Unmark` methods on our implementation of `GCObject`, which will flip the value of the least-significant bit of the method-table pointer. `IsMarked` checks the current value of that bit. Last but not least, we introduce a `MethodTable` property which applies a bit mask to the method-table pointer so that we don't have to worry about whether the object is marked or not whenever we just want to access the method-table (ideally we should only do so in code paths where an object _might_ be marked, but we don't really care about that level of performance at this point).

```csharp
[StructLayout(LayoutKind.Sequential)]
public unsafe struct GCObject
{
    public MethodTable* RawMethodTable;
    public uint Length;

    public readonly MethodTable* MethodTable => (MethodTable*)((nint)RawMethodTable & ~1);

    public bool IsMarked() => ((nint)RawMethodTable & 1) != 0;

    public void Mark() => RawMethodTable = (MethodTable*)((nint)MethodTable | 1);

    public void Unmark() => RawMethodTable = (MethodTable*)((nint)MethodTable & ~1);

    // ...
}
```

Great, now we can implement our actual traversal of the reference tree! We use a DFS (depth-first search) instead of a BFS (breadth-first search) under the assumption that we are more likely to sequentially scan objects that are related to each other, which should improve cache locality. We must be careful to not use recursion, as the object graph can get really big.

```csharp
private void ScanRoots(GCObject* obj, ScanContext* context, GcCallFlags flags)
{
    if ((IntPtr)obj == 0)
    {
        return;
    }

    if (flags.HasFlag(GcCallFlags.GC_CALL_INTERIOR))
    {
        // TODO
        return;
    }

    _markStack.Push((IntPtr)obj);

    while (_markStack.Count > 0)
    {
        var ptr = _markStack.Pop();
        var o = (GCObject*)ptr;

        if (o->IsMarked())
        {
            continue;
        }

        o->EnumerateObjectReferences(_markStack.Push);
        o->Mark();
    }
}
```

Notice that for now we ignore roots that are marked with the `GC_CALL_INTERIOR` flag. Those are _interior pointers_. Consider for instance the following code:

```csharp
private static ref int GetInteriorPointer(int size)
{
    var array = new int[size];
    return ref array[5];
}
```

`GetInteriorPointer` returns a reference to an int, but that int is stored inside of an array. If the array is ever collected while some code holds that reference, bad things will happen. Therefore, by some mechanism, that `ref int` (called interior pointer) must keep the whole array alive. This is challenging for the GC because we are handed a pointer to an arbitrary part of an object, and we must somehow find the beginning of that object to mark it. For now, we're going to just pretend that this problem doesn't exist, and ignore interior pointers entirely.

# Sweeping

After we marked the objects that are referenced, directly or indirectly, by the roots, we need to do a full scan of the heap. Whenever we find an object that isn't marked, we know that this object isn't reachable anymore and we can clear it. For now, our GC doesn't reuse memory so we're just going to clear the memory to make sure that user applications will crash if we accidentally collect an object that is still reachable. To keep the heap in a walkable state, we replace the old object with a free object (free objects are explained [in part 4](https://minidump.net/writing-a-net-gc-in-c-part-4/)). Also, when we find a marked object, we make sure to unmark it to restore its method-table pointer into its original state.

The `WalkHeapObjects` is the same logic [as the `TraverseHeap` method of part 4](https://minidump.net/writing-a-net-gc-in-c-part-4/), but cleaned up and rewritten into an enumerator to be easily reusable.

```csharp
private void Sweep()
{
    foreach (IntPtr ptr in WalkHeapObjects())
    {
        var obj = (GCObject*)ptr;

        bool marked = obj->IsMarked();
        obj->Unmark();

        bool isFreeObject = obj->MethodTable == _freeObjectMethodTable;

        if (!marked && !isFreeObject)
        {
            var startPtr = ptr - IntPtr.Size; // Include the header
            var endPtr = Align(startPtr + (nint)obj->ComputeSize());

            // Clear the memory
            new Span<byte>((void*)startPtr, (int)(endPtr - startPtr)).Clear();

            // Allocate a free object to keep the heap walkable
            AllocateFreeObject(ptr, (uint)(endPtr - startPtr - SizeOfObject));
        }
    }
}

private IEnumerable<IntPtr> WalkHeapObjects()
{
    foreach (var segment in _segments)
    {
        foreach (var obj in WalkHeapObjects(segment.Start, segment.Current))
        {
            yield return obj;
        }
    }
}

private IEnumerable<IntPtr> WalkHeapObjects(nint start, nint end)
{
    var ptr = start + IntPtr.Size;

    while (ptr < end)
    {
        yield return ptr;
        ptr = FindNextObject(ptr);
    }

    static unsafe nint FindNextObject(nint current)
    {
        var obj = (GCObject*)current;
        return Align(current + (nint)obj->ComputeSize());
    }
}
```

And that's it! Of course, if you try to run an application with the GC at this stage, it will quickly crash: we have yet to implement support for interior pointers, and we have completely skipped the other two types of roots: GC handles and the finalization queue. This will be the subject of our next articles.

As usual, the full code is available on [GitHub](https://github.com/kevingosse/ManagedDotnetGC/).

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
