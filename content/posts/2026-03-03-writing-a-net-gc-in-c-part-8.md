---
url: writing-a-net-gc-in-c-part-8
title: 'Writing a .NET Garbage Collector in C#  -  Part 8: Interior pointers'
subtitle: Using NativeAOT to write a .NET GC in C#. This time, we look at what interior pointers are and why they're so challenging for the GC. We also introduce the concept of brick table.
summary: Using NativeAOT to write a .NET GC in C#. This time, we look at what interior pointers are and why they're so challenging for the GC. We also introduce the concept of brick table.
date: 2026-03-03
tags:
- dotnet
- nativeaot
- garbage-collection
author: Kevin Gosse
thumbnailImage: /images/2026-03-03-writing-a-net-gc-in-c-part-8-1.png
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

In the past two articles, we've been looking at how to implement the mark phase of our garbage collector. One subject that I conveniently left out is interior pointers.

If you recall, our `ScanRoots` method looks like this:

```csharp
private void ScanRoots(GCObject* root, ScanContext* context, GcCallFlags flags)
{
    if ((IntPtr)root == 0)
    {
        return;
    }

    if (flags.HasFlag(GcCallFlags.GC_CALL_INTERIOR))
    {
        // TODO
        return;
    }

    _markStack.Push((IntPtr)root);

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

Today, we'll address that to-do, and you'll quickly understand why it deserves an article of its own.

In case you need a refresher, don’t hesitate to jump back to those past articles:

- [Part 1:](https://minidump.net/2025-28-01-writing-a-net-gc-in-c-part-1/) Introduction and setting up the project
- [Part 2:](https://minidump.net/writing-a-net-gc-in-c-part-2/) Implementing a minimal GC
- [Part 3:](https://minidump.net/writing-a-net-gc-in-c-part-3/) Using the DAC to inspect the managed objects
- [Part 4:](https://minidump.net/writing-a-net-gc-in-c-part-4/) Walking the managed heap
- [Part 5:](https://minidump.net/writing-a-net-gc-in-c-part-5/) Decoding the GCDesc to find the references of a managed object
- [Part 6:](https://minidump.net/writing-a-net-gc-in-c-part-6/) Implementing the mark and sweep phases
- [Part 7:](https://minidump.net/writing-a-net-gc-in-c-part-7/) Marking handles


# What are interior pointers?

As suggested by the name, interior pointers are pointers to the interior of an object. Typically, a reference points to the beginning of an object. Or more specifically, just past its header:

```csharp
var obj = new MyObject();
```

```b                                     
                     MyObject
                +-----------------+  
                | Object header   |  
                +-----------------+  
       obj ---> | MethodTable*    | 
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

An interior pointer will point to somewhere within the object:

```csharp
var obj = new MyObject();
ref var ptr = ref obj.Field2;
```

```b                                     
                     MyObject  
                +-----------------+  
                | Object header   |  
                +-----------------+  
       obj ---> | MethodTable*    | 
                +-----------------+ 
                |  Field1         | 
                +-----------------+ 
       ptr ---> |  Field2         | 
                +-----------------+ 
                |  Field3         |
                +-----------------|
                |  Field4         |
                +-----------------+
```

It's fairly rare to have an interior pointer pointing to a field of an object, but getting a reference to an element of an underlying array is becoming more and more common in modern high-performance C# code. For instance, imagine a dictionary storing counters:

```csharp
private Dictionary<string, int> _counters = new();

public void Increment(string key)
{
    _counters.TryGetValue(key, out var value);
    value++;
    _counters[key] = value;
}
```

We have to get the value (if it exists), increment it, then store it back. That's two dictionary lookups. Instead, we can do:

```csharp
private Dictionary<string, int> _counters = new();

public void Increment(string key)
{
    ref var value = ref CollectionsMarshal.GetValueRefOrAddDefault(_counters, key, out _);
    value++;
}
```

`CollectionsMarshal.GetValueRefOrAddDefault` returns a reference to the item in the underlying array, and we can then directly mutate it, saving one lookup.

But there is an even simpler and more common use-case: spans.

```csharp
long[] array = [1, 2, 3, 4];
var span = array.AsSpan();
var slice = span.Slice(2);
```

Under the hood, spans work by keeping a direct reference to the interior of the array:

```a
                      long[]         
                +-----------------+  
                | Object header   |  
                +-----------------+  
     array ---> | MethodTable*    | 
                +-----------------+ 
                |  Length         | 
                +-----------------+ 
                |  Padding        | 
                +-----------------+ 
      span ---> |  1              |
                +-----------------|
                |  2              |
                +-----------------|
     slice ---> |  3              |
                +-----------------|
                |  4              |
                +-----------------+
```

Anyway, at this point you get it: a reference does not necessarily point to the beginning of an object. But how is that a problem?

# Interior pointers from the GC perspective

Interior pointers, just like normal references, must keep the object alive. Consider for instance this code:

```csharp
public static void Test()
{
    long[] array = [1, 2, 3, 4];
    var span = array.AsSpan().Slice(2);
    DoStuff(span);
}

private static void DoStuff(ReadOnlySpan<long> span)
{
    // Do something with the span
}
```

By the time `DoStuff` is invoked, the `array` reference is no longer considered live by the JIT, and thus is eligible for garbage collection. It is necessary for spans, and therefore interior pointers, to keep the object alive just like normal references, otherwise we may end up reading invalid memory.

When marking objects, the GC needs to read the method-table pointer, for at least two reasons:
 - We need the GCDescs, stored next to the method-table, to find the offsets of the outgoing references of the object. See [part 5](https://minidump.net/writing-a-net-gc-in-c-part-5/) for a refresher on the topic.
 - We use the least-significant bit of the method-table pointer to mark the object, as explained in [part 6](https://minidump.net/writing-a-net-gc-in-c-part-6/).

For normal references, this is trivial for the GC because it directly receives the address of the method-table pointer. For interior pointers, it must somehow backtrack to find the beginning of the object.

```b
                |                 |
                |                 |
                |                 |
                |       ???       |
                |                 |
                |                 |
                +-----------------|
       ptr ---> |  3              |
                +-----------------|
                |  4              |
                +-----------------+
```

There is nothing fundamentally problematic in reading memory backwards, but the problem is to know when to stop. How can the GC know when it has found the beginning of the object? One tempting solution could be to look for anything that points to a valid method-table. However, this is prone to false positives. For instance, this code:

```csharp
public class MyObject
{
    public nint Handle;
}

var obj = new MyObject { Handle = typeof(string).TypeHandle.Value };
```

`TypeHandle.Value` is the address of the method-table, and therefore an interior pointer to `MyObject` would fool this heuristic. Arguably, this scenario is completely artificial and extremely unlikely to ever happen in a real application, but even a simple hashcode could end up colliding with the address of a method-table. And the problem is impossible to solve: no matter what complex pattern you may imagine to identify the beginning of an object, there is always a non-zero probability that this same pattern appears in randomly generated data.

What can the GC do then? Without additional context, the only safe way to identify the beginning of an object is to start from the beginning of the memory segment and [walk the heap](https://minidump.net/writing-a-net-gc-in-c-part-4/) up to the given address. This is of course completely unacceptable from a performance perspective, we can't possibly scan an entire memory segment (4MB for a basic region in the .NET GC, and up to 4GB before regions were introduced!) every time we need to mark an interior pointer.

We need some kind of trick to avoid having to scan the whole segment every time. The trick used by the .NET GC, and that we're going to implement too, is called a brick table.

# Designing a brick table

The core idea is simple: building a map of where the objects are located in memory, so we can quickly find what object a given address belongs to. There are many different ways of building a brick table, and every implementation is a trade-off between precision and memory usage. To better understand the trade-off, let's consider a few practical examples.

First, let's imagine an exhaustive brick table. It keeps track of the position of every object on the heap. One way to implement it is with a bitmap, with each bit tracking 1 byte of memory: if the bit is set, it means there is an object starting on that byte. For a heap of 1 GB, our brick table would use 128 MB of memory. That's a lot of wasted RAM! Fortunately, we can optimize it by noticing that all objects are aligned on a pointer-size boundary. So on x64, every object starts on an address that is a multiple of 8. Taking that into account, it means that every bit on our brick table can track 8 bytes without the risk of missing an object. It now uses 128 / 8 = 16 MB of memory. It might be acceptable in some scenarios, but that's still a lot.

To shrink it further, we need to accept that the brick table won't be exhaustive: instead of giving us the exact address of the object referenced by the interior pointer, it will give us the address of an object nearby, and we will have to walk the heap a bit to find the right object. If we say, for instance, that every bit tracks 64 bytes, we reduce the size of the brick table to 2 MB. However, the bitmap approach shows its limits: since we need to know exactly where an object begins, we would only be able to mark objects that are aligned on a 64 bytes boundary, thus greatly reducing the efficiency of the brick table. If it's hard to picture, check this representation:

```b
----------------------------------------
        |        |        |        |
----------------------------------------
        0        0        0        0
```

Every bar `|` indicates a 64 bytes boundary, and is mapped to a single bit in the bitmap. If an object (`*`) is allocated at the beginning of the boundary, we can set the bit to indicate its position:

```b
----------------------------------------
        |*       |        |        |
----------------------------------------
        1        0        0        0
```

So when scanning the brick table, we know that a bit set means that an object is located at the beginning of that chunk of 64 bytes of memory.

However, if the object is located somewhere in the middle:

```b
----------------------------------------
        |     *  |        |        |
----------------------------------------
        0        0        0        0
```

Then we can't mark the bit. Otherwise, when we scan the bitmap and see a bit set, we would only know that an object is _somewhere_ in that 64 bytes range, and not exactly where. Then we circle back to the very problem that we're trying to solve: how do we identify where an object begins?

The bottom line is that the bitmap approach works pretty well when trying to map the objects exhaustively, but breaks down when we try to reduce the precision to save memory.

An approach that scales better is to implement the brick table as an array of offsets. Each element in the array indicates the offset of the first object in a region of memory (or the last, depending on how we choose to implement it). If we implement it as an array of bytes, then each element covers a region of 255 bytes of memory (not 256 because the value '0' is reserved to represent 'no object in that range').
So, let's imagine that the beginning of our brick table is: `[0x54, 0x0, 0x22, 0x23]`. After decoding it, we know that:
 - There is an object at the address `0xFF*0 + 0x54 = 0x54`
 - There is no object in the range `[0xFF*1, 0xFF*2) = [0xFF, 0x1FE)`
 - There is an object at the address `0xFF*2 + 0x22 = 0x220`
 - There is an object at the address `0xFF*3 + 0x23 = 0x320`

 With that information, if we encounter an interior pointer pointing at 0x300, we know there is an object at 0x220 and we can start walking the heap from there.

 This implementation covers 255 bytes of memory for every byte, so we would need 4 MB to cover 1 GB of heap. It gets even better if we consider as before that all objects are aligned on an 8 byte boundary, meaning that every byte in the brick table covers 255*8 = 2040 bytes of memory. So we would need just ~500 KB to cover 1 GB of heap. That's very efficient!

 To perform a lookup in that brick table, we apply the following algorithm:
  - Given an address, we compute what byte of the brick table covers that address (by dividing it by 255*8)
  - We read that byte. If it's non-zero, then we have the address of the first object in that range and we can start walking the heap from there
  - If it's zero, then we read the previous entry of the brick table, and we repeat until we find a non-zero entry or we reach the beginning (in which case we know we have to walk the whole segment)

  To avoid that last step and not have to scan an arbitrary number of entries backward, the .NET GC has implemented another optimization. It uses signed shorts for the brick table entries. A positive value indicates the offset of the object, and a negative value indicates how far back in the brick table we must jump to find a positive entry. So the algorithm becomes:

  - Given an address, we compute what entry of the brick table covers that address
  - We read that entry. If it's greater than zero, then we have the address of the first object in that range and we can start walking the heap from there
  - If it's negative, we adjust the brick index by adding the (negative) value, and repeat the lookup. For instance, let's say we started from the 9th entry in the table, and we read the value -4. Then we jump back to entry 5 and we're guaranteed to find a non-zero value.

# The implementation

For my GC, I decided to not implement the optimization done by the .NET GC (at least for now) and have every byte cover 255*8 = 2040 bytes of memory. I allocate the brick table at the beginning of each of my segments. My segments are 4 MB, so the brick tables use roughly 2 KB of memory each. I also decided to store the offset of the last object in the range instead of the first, because otherwise the first byte of the brick table is wasted (since the first object of a segment will always be allocated at the beginning of the segment). I'm not going to explain the code in detail because it's just boring computations without much added value. If you're curious regardless, you can check the source code [directly on GitHub](https://github.com/kevingosse/ManagedDotnetGC/blob/master/ManagedDotnetGC/Segment.cs).

On my `Segment` object, I exposed two APIs: `void MarkObject(IntPtr addr)`, which marks a given address on the brick table, and `IntPtr FindClosestObjectBelow(IntPtr addr)` which, given an address, returns the address of the closest known object below that address. This is what we're going to use to mark our interior pointers:

```csharp
private void ScanRoots(GCObject* root, ScanContext* context, GcCallFlags flags)
{
    if ((IntPtr)root == 0)
    {
        return;
    }

    if (flags.HasFlag(GcCallFlags.GC_CALL_INTERIOR))
    {
        // Find the segment containing the interior pointer
        var segment = _segmentManager.FindSegmentContaining((nint)root);

        if (segment.IsNull)
        {
            Write($"  No segment found for interior pointer {(IntPtr)root:x2}");
            return;
        }

        var objectStartPtr = segment.FindClosestObjectBelow((IntPtr)root);
        bool found = false;

        // Walk the heap starting from objectStartPtr,
        // until we find the object pointed at by the interior pointer
        foreach (var ptr in WalkHeapObjects(objectStartPtr, (IntPtr)root))
        {
            var o = (GCObject*)ptr;
            var size = o->ComputeSize();

            // Is the interior pointer within this object?
            if ((IntPtr)o <= (IntPtr)root && (IntPtr)root < (IntPtr)o + (nint)size)
            {
                root = o;
                found = true;
                break;
            }
        }

        if (!found)
        {
            Write($"  No object found for interior pointer {(IntPtr)root:x2}");
            return;
        }
    }

    _markStack.Push((IntPtr)root);

    while (_markStack.Count > 0)
    {
        var ptr = _markStack.Pop();
        var o = (GCObject*)ptr;

        if (o->IsMarked())
        {
            continue;
        }

        var segment = _segmentManager.FindSegmentContaining((nint)o);

        if (segment.IsNull)
        {
            continue;
        }

        o->EnumerateObjectReferences(_markStack.Push);
        o->Mark();
        segment.MarkObject((IntPtr)o); // Update the brick table
    }
}
```

As you can see, two things have changed in the `ScanRoots` method:
 - We now properly handle interior pointers, by using the brick table to get the address of an object below the target address, then walking the heap until we find the right object
 - When we mark an object, we also mark its location in the brick table

Currently, the GC only sweeps memory and [replaces the dead object with a special free object](https://minidump.net/writing-a-net-gc-in-c-part-6/) which is walkable like a normal object. This means that we don't need to unmark objects from the brick table for now, but we'll have to revisit that in the future when we implement an allocator that is capable of reusing those free spaces.

Also, note that during the first GC, the brick table is empty. It means that scanning interior pointers will be especially expensive during that time. To reduce this cost, I update the brick table whenever we find objects during the mark phase, so it gets populated progressively. But we can do more: threads are mostly autonomous when allocating objects, thanks to their [allocation context](https://minidump.net/writing-a-net-gc-in-c-part-2/), but they still call the `Alloc` API of the GC whenever they need a new allocation context. [We can update the brick table when that happens](https://github.com/kevingosse/ManagedDotnetGC/blob/500772ac383195177d74c779e554e8861a28f24e/ManagedDotnetGC/GCHeap.cs#L200).

With that, our GC is finally able to resolve interior pointers! As you can see, they're a lot harder to deal with than it first seems. And it's very important that the GC is as efficient as possible when scanning them, because they're typically used in highly optimized code where performance matters.

You might have noticed that, because of the brick table, our GC now has to frequently retrieve the segment owning a given address (done with the `_segmentManager.FindSegmentContaining` method). The method is currently very inefficient (it just iterates over the list of segments until it finds the right one) so it will become a bottleneck. Because of this, and because of the topic of frozen segments that we entirely ignored so far, we're going to completely rethink the way memory is allocated by our GC. This will be the subject of the next article.

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
