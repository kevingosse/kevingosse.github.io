---
url: writing-a-net-gc-in-c-part-9
title: 'Writing a .NET Garbage Collector in C#  -  Part 9: Frozen segments and new allocation strategy'
subtitle: Using NativeAOT to write a .NET GC in C#. In this part, we look at what the GC must do to properly handle frozen segments, and we change the allocation strategy to make it more efficient.
summary: Using NativeAOT to write a .NET GC in C#. In this part, we look at what the GC must do to properly handle frozen segments, and we change the allocation strategy to make it more efficient.
date: 2026-03-17
tags:
- dotnet
- nativeaot
- garbage-collection
author: Kevin Gosse
thumbnailImage: /images/2026-03-17-writing-a-net-gc-in-c-part-9-2.png
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

One great thing about writing my own GC is that it allows me to better understand some design decisions of the .NET GC by experiencing firsthand some of the problems they are meant to solve. Sure, I spent two years studying the source code of the GC for the second edition of Pro .NET Memory Management, but it taught me the _how_, not the _why_.

One such case is the frozen segments. One year ago, I wrote [an article explaining how to allocate objects into frozen segments](https://minidump.net/exploring-frozen-segments/). Back then, I knew you had to call `RegisterFrozenSegment` to declare the frozen segments to the GC, to be allowed to allocate objects in them. However, I didn't understand _why_ that was needed, or what would happen if you actually didn't.

As you can see in the response I wrote to one of the comments under the article:

{{<image classes="fancybox center" src="/images/2026-03-17-writing-a-net-gc-in-c-part-9-1.png" >}}

Well, now I understand, and soon you will too.

In case you need a refresher before jumping in, don't hesitate to read or re-read those past articles:

- [Part 1:](https://minidump.net/2025-28-01-writing-a-net-gc-in-c-part-1/) Introduction and setting up the project
- [Part 2:](https://minidump.net/writing-a-net-gc-in-c-part-2/) Implementing a minimal GC
- [Part 3:](https://minidump.net/writing-a-net-gc-in-c-part-3/) Using the DAC to inspect the managed objects
- [Part 4:](https://minidump.net/writing-a-net-gc-in-c-part-4/) Walking the managed heap
- [Part 5:](https://minidump.net/writing-a-net-gc-in-c-part-5/) Decoding the GCDesc to find the references of a managed object
- [Part 6:](https://minidump.net/writing-a-net-gc-in-c-part-6/) Implementing the mark and sweep phases
- [Part 7:](https://minidump.net/writing-a-net-gc-in-c-part-7/) Marking handles
- [Part 8:](https://minidump.net/writing-a-net-gc-in-c-part-8/) Interior pointers

# The NonGC heap

First, what are we even talking about? The GC allows managed objects to be allocated in segments of unmanaged memory known as _frozen segments_. Together, these frozen segments form the NonGC heap.

What's the point? The NonGC heap is intended to store objects that are immortal and don't reference other objects (or at least, don't reference non-immortal objects). Scanning those objects during garbage collection would be a waste of resources, as they can't be collected and they don't keep other objects alive. The NonGC heap is the way to tell the GC to ignore them.

Since .NET 8, the JIT is aware of the NonGC heap, and takes advantage of it to implement some optimizations. For instance, string literals are stored in the NonGC heap, which allows the JIT to directly embed the address of those objects in the assembly code when compiling a function, skipping a layer of indirection. A few examples are given [in the official documentation](https://github.com/dotnet/runtime/blob/main/docs/design/features/NonGC-Heap.md).

# Frozen segments and the GC

The GC API declares a few functions that the GC must implement to handle frozen segments: `RegisterFrozenSegment`, `UnregisterFrozenSegment`, `UpdateFrozenSegment`, and `IsInFrozenSegment`. `RegisterFrozenSegment` and `UnregisterFrozenSegment` are self-explanatory. `UpdateFrozenSegment` is used to tell the GC which portion of the frozen segment is currently filled with objects. `IsInFrozenSegment` is mostly called by the JIT to determine whether a given object is managed by the GC.

When I first implemented my GC, I wrote a very basic implementation of those methods:

```csharp
partial class GCHeap
{
    private readonly ConcurrentDictionary<nint, StrongBox<(nint start, nint end)>> _frozenSegments = new();
    private int _frozenSegmentIndex;

    public unsafe nint RegisterFrozenSegment(segment_info* pseginfo)
    {
        var handle = Interlocked.Increment(ref _frozenSegmentIndex);
        _frozenSegments.TryAdd(handle, new((pseginfo->pvMem + pseginfo->ibFirstObject, pseginfo->pvMem + pseginfo->ibAllocated)));

        return handle;
    }

    public void UnregisterFrozenSegment(nint seg)
    {
        _frozenSegments.TryRemove(seg, out _);
    }

    public unsafe bool IsInFrozenSegment(GCObject* obj)
    {
        foreach (var segment in _frozenSegments.Values)
        {
            if ((nint)obj >= segment.Value.start && (nint)obj < segment.Value.end)
            {
                return true;
            }
        }

        return false;
    }

    public void UpdateFrozenSegment(nint seg, nint allocated, nint committed)
    {
        if (_frozenSegments.TryGetValue(seg, out var segment))
        {
            segment.Value.end = allocated;
        }
    }
}
```

It keeps track of the frozen segments in a `ConcurrentDictionary`, just to be able to answer to whoever calls `IsInFrozenSegment`. Nothing else. After all, isn't the whole point of the frozen segments that the GC would ignore them? It felt very odd to me at that time: can't the runtime do this by itself? What's the point of delegating this bookkeeping to the GC?

However, I later discovered that this simple program crashed:

```csharp
string str = "Hello";

InnerFunction(str);

static void InnerFunction(object obj)
{
    GC.Collect();
    Console.WriteLine(obj is string);
}
```

Why is that? 

To understand, let's recap how our GC works. During mark phase, we enumerate the roots and walk the reference tree to discover all the objects that are reachable. When we find an object, we mark it [by setting the least-significant bit of the method-table pointer to 1](https://minidump.net/writing-a-net-gc-in-c-part-6/). Later, during the sweep phase, we walk the heap and, for each object:

 - If the object is not marked, consider it as free space
 - If the object is marked, unmark it, to restore its original method-table pointer

It sounds like it should work, and it did during my initial tests. However, let's consider what happens in the previous example:

 - `Hello` is a string literal, it's allocated by the runtime in a frozen segment
 - A reference to that string literal is given to the `InnerFunction` method
 - Inside `InnerFunction`, a garbage collection is triggered
 - During mark phase, the GC enumerates the roots and finds `obj`. It marks the object in the frozen segment, thus altering its method-table pointer
 - During sweep phase, the GC walks the heap and unmarks all objects that are still alive. **However, the string literal lives outside of the managed heap, so the GC doesn't find it during the walk**
 - The garbage collection ends and the runtime execution resumes. `obj is string` checks the method-table of the object, and triggers the crash because the method-table pointer is invalid

 That's bad. How can we fix that? A straightforward way is to walk the frozen segments, in addition to the managed heap, at the end of the sweep phase to unmark the objects:

 ```csharp
private void UnmarkFrozenSegments()
{
    foreach (var segment in _frozenSegments.Values)
    {
        foreach (var obj in WalkHeapObjects(segment.Value.start, segment.Value.end))
        {
            var o = (GCObject*)obj;
            o->Unmark();
        }
    }
}
 ```

 With this simple fix, the program stops crashing. However, it means that the GC has to walk the frozen segments just like "normal" managed segments, which defeats some of their purpose. On paper, one of the main benefits of frozen segments was precisely that the GC could ignore them. Can we do better?
 
If we don't want to have to unmark the objects in the frozen segments, then we shouldn't mark them to begin with. We could tweak the mark phase with an additional check:

```csharp
private void ScanRoots(GCObject* obj, ScanContext* context, GcCallFlags flags)
{
    // ... (cut for brevity)

    _markStack.Push((IntPtr)obj);

    while (_markStack.Count > 0)
    {
        var ptr = _markStack.Pop();
        var o = (GCObject*)ptr;

        if (o->IsMarked())
        {
            continue;
        }

        // Only mark the object if it's in the managed heap
        if (!IsInFrozenSegment(o))
        {
          o->EnumerateObjectReferences(_markStack.Push);
          o->Mark();
        }
    }
}
```
While that would work in theory, this is a bad trade-off: `IsInFrozenSegment` enumerates all the frozen segments to determine whether the given address belongs to one of them. This is way too expensive to be called on every referenced object. We could lower the complexity to `O(log n)` by using interval trees, but that's still too much. Alternatively, we could use bitmaps like described [in the previous article](https://minidump.net/writing-a-net-gc-in-c-part-8/) to find out if a given address belongs to a frozen segment in `O(1)` time. But because frozen segments are not managed by the GC, they could be allocated anywhere in memory and be of any size. So our bitmap would have to map the entire address space, wasting a lot of memory.

Or we could flip the problem around.

# What if I had infinite memory?

As I just mentioned, knowing if an address belongs to a frozen segment is a difficult problem to solve (in an efficient way) because we don't control where frozen segments are allocated. However, we do have complete control over where managed segments are allocated, so instead of answering the question "does it belong to a frozen segment?", we can focus on the question "does it _not_ belong to a managed segment?".

To understand where we're going, we must first recall some basic concepts about memory management. Have you ever tried checking how much memory a simple .NET console app uses?

```csharp
Console.WriteLine("Hello, World!");

var currentProcess = System.Diagnostics.Process.GetCurrentProcess();

Console.WriteLine($"Private memory: {currentProcess.PrivateMemorySize64 / 1024 / 1024} MB");
Console.WriteLine($"Virtual memory: {currentProcess.VirtualMemorySize64 / 1024 / 1024} MB");
```

{{<image classes="fancybox center" src="/images/2026-03-17-writing-a-net-gc-in-c-part-9-2.png" >}}

6 MB of private memory, that's probably a lot for a simple Hello World, but not so bad for a managed language. And virtual memory is... _Wait... How many digits is that?_ What do you mean _2,297,939 MB_?

{{<image classes="fancybox center" src="/images/2026-03-17-writing-a-net-gc-in-c-part-9-3.jpg" >}}

Of course there is a trick, and the trick is that it's all _reserved_ memory.

As you probably know, on a modern OS (modern as in _anything since at least the mid 90s_), each process gets its own address space. If process A writes a value to the address `0x1000`, process B will see a completely different value. Each process is given access to _virtual_ memory, and the OS is in charge of mapping it to real, physical memory, as needed. It works in two phases:
 - First, the process _reserves_ some memory. It tells the OS: "I plan to use the memory in that range". At that point, the OS isn't doing much, just noting that information in whatever bookkeeping structure it uses internally
 - Then, the process _commits_ memory. At that point, it wants to write actual data to the reserved memory, so the OS has to map the range to actual physical memory.

 This is what we see in our console application: it has reserved 2 TB (virtual memory), but committed only 6 MB (private memory). Why reserve so much memory if it's going to use only a tiny fraction of it? Well, precisely to solve the problem that we mentioned earlier.

 On 64-bit, the address space is huge, practically infinite when compared to what applications actually require. This is convenient because infinity minus a big chunk is still infinity. The trick is therefore to reserve a huge chunk of memory upfront, big enough to cover the needs of any reasonably sized app. Then we commit some of that memory when we actually use it. This way, we get the guarantee that all the managed memory sits within a well-defined address range, and conversely that anything outside of that range is unmanaged memory.

With this strategy, checking if an object might belong to a frozen segment becomes a simple test against a lower and higher bound. Of course, it only works on 64-bit. On 32-bit, the address space is just 4 GB, and some of it must be saved for native allocations. In the .NET garbage collector, there is a surprising amount of complexity just in the code implementing the strategy to reserve memory. The GC tries its hardest to keep all its allocations in a contiguous chunk of reserved memory, but has many fallbacks in case it fails, and in the worst case it will revert to walking frozen segments to unmark objects at the end of the garbage collection, like described in the previous part. For this custom GC, we're not going to bother with any of that, and from that point onwards this GC will officially only support 64-bit runtimes.

{{< alert >}}
This article makes it sound like frozen segments are the main reason why the .NET GC tries to keep all of its allocations within a single chunk of contiguous reserved memory. There are actually several other reasons, such as reducing the size of the card table by keeping all objects within a compact address range. The frozen segment issue simply happened to be the first problem we encountered because of the particular order in which I chose to implement the features of our GC.
{{< /alert >}}

# The new allocation strategy

Enough talk, it's time to implement the new allocation strategy. Until now, our GC has relied on [`NativeMemory.Alloc`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.nativememory.alloc?view=net-10.0&WT.mc_id=DT-MVP-5003493). Internally, it is bound to the C `malloc` API, so there was a big risk that our allocations would be interleaved with native allocations from other components. We're going to replace it with our own `NativeAllocator`. It works by immediately reserving 2 TB of memory, then committing chunks of it on demand to satisfy allocations. Both operations use the Windows [`VirtualAlloc`](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualalloc?WT.mc_id=DT-MVP-5003493) API (or [mmap](https://man7.org/linux/man-pages/man2/mmap.2.html) on Linux). If we exhaust the whole 2 TB, we should in theory degrade to a "fragmented address space" mode, but for this pet project I'm not going to bother: _2 TB ought to be enough for anybody_.

In the constructor, we immediately reserve the memory, which gives us the lower and upper bounds within which all managed memory will live:

```csharp
internal partial class NativeAllocator : IDisposable
{
    private const uint MEM_COMMIT = 0x00001000;
    private const uint MEM_RESERVE = 0x00002000;
    private const uint MEM_DECOMMIT = 0x00004000;
    private const uint MEM_RELEASE = 0x00008000;

    private const uint PAGE_READWRITE = 0x04;

    public NativeAllocator(long size)
    {
        _lowestAddress = VirtualAlloc(IntPtr.Zero, (UIntPtr)size, MEM_RESERVE, PAGE_READWRITE);

        if (_lowestAddress == IntPtr.Zero)
        {
            throw new OutOfMemoryException("Failed to reserve memory");
        }

        _highestAddress = _lowestAddress + (nint)size;
        _nextFreeAddress = _lowestAddress;
    }

    public nint LowestAddress => _lowestAddress;
    public nint HighestAddress => _highestAddress;

    public void Dispose()
    {
        if (_lowestAddress != IntPtr.Zero)
        {
            VirtualFree(_lowestAddress, UIntPtr.Zero, MEM_RELEASE);
            _lowestAddress = IntPtr.Zero;
            _highestAddress = IntPtr.Zero;
        }
    }
}
```

The allocation code is slightly more involved, for two reasons: it needs to be thread-safe, and we can only commit multiples of the OS page size:

```csharp
private static readonly int PageSize = Environment.SystemPageSize;

public nint Allocate(nint size)
{
    if (size <= 0)
    {
        return IntPtr.Zero;
    }

    var alignedSize = (size + (PageSize - 1)) & ~(nint)(PageSize - 1);
    nint address;

    while (true)
    {
        address = Volatile.Read(ref _nextFreeAddress);
        var end = address + alignedSize;

        if (end > HighestAddress)
        {
            throw new OutOfMemoryException("Not enough memory to allocate");
        }

        if (Interlocked.CompareExchange(ref _nextFreeAddress, end, address) == address)
        {
            break;
        }
    }

    var result = VirtualAlloc(address, (UIntPtr)alignedSize, MEM_COMMIT, PAGE_READWRITE);

    if (result == IntPtr.Zero)
    {
        throw new OutOfMemoryException("VirtualAlloc failed to commit memory");
    }

    return address;
}
```

Freeing memory is simpler (at least for now):

```csharp
public void Free(nint address, nint size)
{
    if (address == IntPtr.Zero || _lowestAddress == IntPtr.Zero)
    {
        return;
    }

    if (address < _lowestAddress || address >= _highestAddress)
    {
        throw new InvalidOperationException($"Address {address:x2} is out of reserved range");
    }

    var alignedSize = (size + (PageSize - 1)) & ~(nint)(PageSize - 1);

    if (!VirtualFree(address, (UIntPtr)alignedSize, MEM_DECOMMIT))
    {
        throw new InvalidOperationException("VirtualFree failed to decommit memory");
    }
}
```

That's it! We can now expose an API that checks if an address is within the range owned by the `NativeAllocator`:

```csharp
public bool IsInRange(nint ptr) => ptr >= LowestAddress && ptr < HighestAddress;
```

And we can plug it directly into the mark phase. It's cheap enough to be called for each object:

```csharp
private void ScanRoots(GCObject* obj, ScanContext* context, GcCallFlags flags)
{
    // ... (cut for brevity)

    _markStack.Push((IntPtr)obj);

    while (_markStack.Count > 0)
    {
        var ptr = _markStack.Pop();
        var o = (GCObject*)ptr;

        if (o->IsMarked())
        {
            continue;
        }

        // Only mark the object if it's in the managed heap
        if (_nativeAllocator.IsInRange((nint)o))
        {
          o->EnumerateObjectReferences(_markStack.Push);
          o->Mark();
        }
    }
}
```

Thanks to this, we don't have to walk the frozen segments to unmark the objects at the end of the garbage collection, because we never mark those objects to begin with.

Of course, our allocator is very simple for now because this GC never reuses freed address ranges. Once we start reclaiming and reusing space, we must switch to a fully-featured allocator, with a free-list to keep track of free chunks of memory, and size buckets to prevent fragmentation. Reimplementing `malloc`, basically.


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
