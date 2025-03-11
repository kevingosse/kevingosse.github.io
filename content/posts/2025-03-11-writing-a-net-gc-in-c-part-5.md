---
url: writing-a-net-gc-in-c-part-5
title: Writing a .NET Garbage Collector in C#  -  Part 5
subtitle: Using NativeAOT to write a .NET GC in C#. In the fifth part, we learn how to decode the GCDesc to find the references of a managed object.
summary: Using NativeAOT to write a .NET GC in C#. In the fifth part, we learn how to decode the GCDesc to find the references of a managed object.
date: 2025-03-11
tags:
- dotnet
- nativeaot
- garbage-collection
author: Kevin Gosse
thumbnailImage: /images/2025-03-11-writing-a-net-gc-in-c-part-5-1.png
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

This is the fifth part of our journey to write a .NET garbage collector in C#. In the previous article, we've started laying the groundwork for implementing the mark phase and we learned how to walk the managed heap. Next we will need to learn how to walk the reference graph, so we can mark objects that are reachable and deduce which objects can be collected.

To walk the reference graph, we need two things:
- The list of roots, which are the references that are not collectible (local variables, static fields, GC handles, ...). They are the starting points of the graph traversal.
- The list of references from an object to another object. This is what this article will focus on.

If you missed the previous parts, you can find them here:
- [Part 1:](https://minidump.net/2025-28-01-writing-a-net-gc-in-c-part-1/) Introduction and setting up the project
- [Part 2:](https://minidump.net/writing-a-net-gc-in-c-part-2/) Implementing a minimal GC
- [Part 3:](https://minidump.net/writing-a-net-gc-in-c-part-3/) Using the DAC to inspect the managed objects
- [Part 4:](https://minidump.net/writing-a-net-gc-in-c-part-4/) Walking the managed heap

# The GCDesc

In order to traverse the reference graph, the GC needs to be able to find the references from an object to another object. In other words, it must know what fields might contain a reference and where to find them in memory. This information is stored in a structure known as `GCDesc`.

Each type has its own `GCDesc`. For efficiency, it doesn't describe the full layout of the object, but only the parts that contain references. Furthermore, it indicates the ranges of memory where the references are located, rather than the exact offsets of each individual field. To give a practical example, consider the following object layout:

```b
+-----------------+
|  Field1 (ref)   |
+-----------------+
|  Field2 (ref)   |
+-----------------+
|  Field3         |
+-----------------+
|  Field4 (ref)   |
+-----------------+
```

Let's assume that all fields are 8 bytes long, and they all store references except `Field3`. The information stored in the `GCDesc` would be the ranges of memory where the references are located, so something like: `[0, 16] [24, 32]`. Because the size of the fields storing references is always the same (the size of a pointer), this is enough information to deduce how many references are stored in the object and where they are located.

{{< alert >}}
As we're going to see soon, the `GCDesc` does not actually store the start and end of the ranges, but the start and the size.
{{< /alert >}}

The `GCDesc` is stored next to the method table of the type. Or more precisely, right before it.

```b
                                         +-----------------+
                      Object             |                 |
                +-----------------+      |                 |
                | Object header   |      |  GCDesc         |
                +-----------------+      +-----------------+
Object ref ---> | MethodTable*    | ---> |  MethodTable    |
                +-----------------+      |                 |
                |  Field1         |      |                 |
                +-----------------+      |                 |
                |  Field2         |      |                 |
                +-----------------+      +-----------------+
                |  Field3         |
                +-----------------|
                |  Field4         |
                +-----------------+
```

# The GCDesc encoding

The encoding of the `GCDesc` [is a bit convoluted](https://github.com/dotnet/runtime/blob/main/src/coreclr/gc/gcdesc.h). The first field is the number of `GCDescSeries` (note: the `GCDesc` is growing backwards in memory, so this is the first field when starting from the method table and going backwards). The count is positive in most cases, but will be negative for arrays of value types, which indicates a different encoding.

## GCDescSeries count > 0

When the count is positive, it indicates the number of `GCDescSeries` that follow. Each `GCDescSeries` is a pair of values describing a range of memory where references are stored:

```csharp
[StructLayout(LayoutKind.Sequential)]
internal struct GCDescSeries
{
    public nint Size;
    public nint Offset;

    public void Deconstruct(out nint size, out nint offset)
    {
        size = Size;
        offset = Offset;
    }
}
```

Therefore, with a count of 2, the `GCDesc` would look like this:
```
+-----------------+
|  Size2          |
+-----------------+
|  Offset2        |
+-----------------+
|  Size1          |
+-----------------+
|  Offset1        |
+-----------------+
|  Count          |
+-----------------+
|  MethodTable    |
|                 |
|                 |
|                 |
|                 |
+-----------------+
```

The offset of the `GCDescSeries` is relative to the start of the object (remember, the start of the object is considered to be the method table pointer, not the header). The size of the `GCDescSeries` is really weird because the base size of the object has been substracted from it so we have to add it back. To take a practical example, consider this object:

```
+-----------------+
| Object header   |
+-----------------+
| MethodTable*    |
+-----------------+
|  Field1 (ref)   |
+-----------------+
```

For a 64-bit process, assuming `Field1` is a reference, the `GCDesc` will contain a single `GCDescSeries` with an offset of 8 and a size of -16 (the actual size of the range is 8, but the base size of the object is 24, so the value is encoded as `8 - 24 = -16`). The size is given in bytes, so we have to divide it by the size of a pointer to get the number of references.

Putting all of this together, we can start writing a method that finds the references in an object, handling only the positive case for now:

```csharp
    private static unsafe void EnumerateObjectReferences(GCObject* obj, Action<IntPtr> callback)
    {
        if (!obj->MethodTable->ContainsGCPointers)
        {
            return;
        }

        var mt = (nint*)obj->MethodTable;
        var objectSize = obj->ComputeSize();

        var seriesCount = mt[-1];

        Console.WriteLine($"Found {seriesCount} series");

        if (seriesCount > 0)
        {
            var series = (GCDescSeries*)(mt - 1);

            for (int i = 1; i <= seriesCount; i++)
            {
                var (seriesSize, seriesOffset) = series[-i];
                seriesSize += (int)objectSize;

                Console.WriteLine($"Series {i}: size={seriesSize}, offset={seriesOffset}");

                var rangeStart = (nint*)((nint)obj + seriesOffset);
                var nbRefsInRange = seriesSize / IntPtr.Size;

                for (int j = 0; j < nbRefsInRange; j++)
                {
                    // Found a reference
                    callback(rangeStart[j]);
                }
            }
        }
        else
        {
            // TODO
        }
    }
```

(see [part 4](https://minidump.net/writing-a-net-gc-in-c-part-4/) for the definition of `GCObject` and `MethodTable`, and the implementation of `ComputeSize`)

## Testing

We can test this code in a "classic" .NET application, which is easier to debug than a GC. For that, we create a new console application and copy the `GCObject` and `MethodTable` definitions from the GC project. We add a helper to get the address of a managed object:
```csharp
internal static unsafe class Utils
{    
    public static nint GetAddress<T>(T obj) => (nint)(*(T**)&obj);
}
```

We can now test the `EnumerateObjectReferences` method:

```csharp

class Program
{
    static void Main(string[] args)
    {
        var obj = new ObjectWithReferences();
        var address = Utils.GetAddress(obj);
        EnumerateObjectReferences((GCObject*)address, ptr =>
        {
            Console.WriteLine($"Found reference to {ptr:x2}");
        });
    }
}

internal class ObjectWithReferences
{
    public object Field1 = new();
    public object Field2 = new();
}
```

Of course, this is very unsafe, and it will break if the GC decides to move objects in the middle of the execution, but that's enough for our simple test. This should display something like:

```
Found 1 series
Series 1: size=16, offset=8
Found reference to 1cf8e40b508
Found reference to 1cf8e40b520
```

What about an object with multiple series? To have multiple series, we must design the object in such a way that the references are not contiguous. You might be tempted to write:

```csharp
internal class ObjectWithMultipleSeries
{
    public object Field1 = new();
    public long Field2;
    public object Field3 = new();
}
```

But the output would still indicate a single series:

```
Found 1 series
Series 1: size=16, offset=8
Found reference to 1766ec0b510
Found reference to 1766ec0b528
```

It turns out that the JIT is allowed to reorder fields, and it always tries to group the references together at the beginning of the object, thus ending up with a single series. Then maybe we could use a structure with a fixed layout:

```csharp
[StructLayout(LayoutKind.Sequential)]
internal struct ObjectWithMultipleSeries()
{
    public object Field1 = new();
    public long Field2;
    public object Field3 = new();
}
```

{{< alert >}}
Notice the subtle primary constructor: `internal struct ObjectWithMultipleSeries()`. Writing just `internal struct ObjectWithMultipleSeries` would be illegal and would require an explicit constructor to use field initializers:

```csharp
[StructLayout(LayoutKind.Sequential)]
internal struct ObjectWithMultipleSeries
{
    public object Field1 = new();
    public long Field2;
    public object Field3 = new();

    public ObjectWithMultipleSeries()
    {
    }
}
```
{{< /alert >}}

Because we use a struct, we must update our code to box it, otherwise we won't find the method table:

```csharp
class Program
{
    static void Main(string[] args)
    {
        // Explicit cast to object to box the struct
        var obj = (object)new ObjectWithReferences();
        var address = Utils.GetAddress(obj);
        EnumerateObjectReferences((GCObject*)address, ptr =>
        {
            Console.WriteLine($"Found reference to {ptr:x2}");
        });
    }
}
```

After running it, we... still see a single series.

```csharp
Found 1 series
Series 1: size=16, offset=8
Found reference to 259b6c0b4e8
Found reference to 259b6c0b500
```

What about the `[StructLayout(LayoutKind.Sequential)]` we added earlier? Well it turns out that the JIT ignores the `StructLayout` when the struct contains references, and automatically switches back to `LayoutKind.Auto`. How is it possible to have multiple series then? There is at least two cases where it could happen. The first one is nested structs (the JIT doesn't reorder fields across structs).

```csharp
internal struct NestedStruct()
{
    public object NestedField1 = new();
    public long NestedField2;
}

internal class ObjectWithMultipleSeries
{
    public long Field1;
    public NestedStruct Field2 = new();
    public object Field3 = new();
}
```

Which has this layout (notice that the reference `Field3` is moved to the beginning of the object, then the value types are laid out in order):

```b
+----------------------+
|  Field3 (ref)        |
+----------------------+
|  Field1              |
+----------------------+
|  NestedField1 (ref)  |
+----------------------+
|  NestedField2        |
+----------------------+
```

And will display:

```b
Found 2 series
Series 1: size=8, offset=8
Found reference to 18201c0b560
Series 2: size=8, offset=24
Found reference to 18201c0b548
```

The second case is inheritance (the layout of the base class can't be changed by the child class).

```csharp
internal class BaseClass
{
    public object BaseField1 = new();
    public long BaseField2;
}

internal class ObjectWithMultipleSeries : BaseClass
{
    public object Field1 = new();
}
```

Which has this layout:
```b
+--------------------+
|  BaseField1 (ref)  |
+--------------------+
|  BaseField2        |
+--------------------+
|  Field1 (ref)      |
+--------------------+
```

And will display:

```
Found 2 series
Series 1: size=8, offset=8
Found reference to 2300e80b528
Series 2: size=8, offset=24
Found reference to 2300e80b510
```

Great! We can also check that our code handles arrays of reference types correctly:

```csharp
class Program
{
    static void Main(string[] args)
    {
        ObjectWithMultipleSeries[] obj = [new(), new(), new()];
        var address = Utils.GetAddress(obj);
        EnumerateObjectReferences((GCObject*)address, ptr =>
        {
            Console.WriteLine($"Found reference to {ptr:x2}");
        });
    }
}
```

Which will display:

```
Found 1 series
Series 1: size=24, offset=16
Found reference to 24baa40b518
Found reference to 24baa40b570
Found reference to 24baa40b5c8
```

Note that we will find a single series no matter how many series the underlying object has, because arrays of reference types only contain the references to the objects, not the objects themselves. The layout will always be:

```b
+--------------------+
|  Reference1 (ref)  |
+--------------------+
|  Reference2 (ref)  |
+--------------------+
|  Reference3 (ref)  |
+--------------------+

```

## GCDescSeries count < 0

There is still one case left to handle: when the `GCDescSeries` count is negative, which indicates an array of value types. This case is special because this is the only time the number of series will depend on the number of elements in the array (this doesn't happen with arrays of reference types because, as mentioned before, they only contain the reference to the objects and therefore are made of a single series).

For instance, if we make an array of the `NestedStruct` type defined earlier, the layout will be:

```b
+----------------------+
|  NestedField1 (ref)  |
+----------------------+
|  NestedField2        |
+----------------------+
|  NestedField1 (ref)  |
+----------------------+
|  NestedField2        |
+----------------------+
|  NestedField1 (ref)  |
+----------------------+
|  NestedField2        |
+----------------------+
```

In that case, the `GCDesc` switches to a different encoding. When the count is negative, the absolute value of the count indicates the number of `ValSerieItem` that follow, preceded by a single offset (again, remember we're growing backwards, so "preceded" actually means it's higher in memory). `ValSerieItem` contains two fields: the number of pointers in the range, and the number of bytes to skip to find the next range. The size of each field is half the size of a pointer (the whole `ValSerieItem` fits in a single pointer).

With a count of -2, the `GCDesc` would look like this:
```
+-----------------+
|  ValSerieItem   |
+-----------------+
|  ValSerieItem   |
+-----------------+
|  Offset         |
+-----------------+
|  Count          |
+-----------------+
|  MethodTable    |
|                 |
|                 |
|                 |
|                 |
+-----------------+
```

`ValSerieItem` is annoying to represent in C# because there is no built-in type to store half a pointer. Instead, we can use a single `nint` as underlying storage, then bit masks to extract the values:

```csharp
internal struct ValSerieItem(nint value)
{
    public uint Nptrs => IntPtr.Size == 4 ? (ushort)(value & 0xFFFF) : (uint)(value & 0xFFFFFFFF);

    public uint Skip => IntPtr.Size == 4 ? (ushort)((value >> 16) & 0xFFFF) : (uint)(((long)value >> 32) & 0xFFFFFFFF);
}
```

To find the references in an array of value types, we must start at the offset, then for each `ValSerieItem` we must read the given number of pointers and jump by `Skip` bytes. This must be repeated as many times as there are elements in the array. The resulting code looks like:

```csharp
    private static unsafe void EnumerateObjectReferences(GCObject* obj, Action<IntPtr> callback)
    {
        if (!obj->MethodTable->ContainsGCPointers)
        {
            return;
        }

        var mt = (nint*)obj->MethodTable;
        var objectSize = obj->ComputeSize();

        var seriesCount = mt[-1];

        Console.WriteLine($"Found {seriesCount} series");

        if (seriesCount > 0)
        {
            // ...
        }
        else
        {
            var offset = mt[-2];
            var valSeries = (ValSerieItem*)(mt - 2) - 1;

            Console.WriteLine($"Offset {offset}");

            // Start at the offset
            var ptr = (nint*)((nint)obj + offset);

            // Retrieve the length of the array
            var length = obj->Length;

            // Repeat the loop for each element in the array
            for (int item = 0; item < length; item++)
            {
                for (int i = 0; i > seriesCount; i--)
                {
                    // i is negative, so this is going backwards
                    var valSerieItem = valSeries + i;

                    Console.WriteLine($"ValSerieItem: nptrs={valSerieItem->Nptrs}, skip={valSerieItem->Skip}");

                    // Read valSerieItem->Nptrs pointers
                    for (int j = 0; j < valSerieItem->Nptrs; j++)
                    {
                        callback(*ptr);
                        ptr++;
                    }

                    // Skip valSerieItem->Skip bytes
                    ptr = (nint*)((nint)ptr + valSerieItem->Skip);
                }
            }
        }
    }
```

If we run the code with an array of three `NestedStruct`, we get:

```b
Found -1 series
Offset 16
ValSerieItem: nptrs=1, skip=8
Found reference to 1de5c80b530
ValSerieItem: nptrs=1, skip=8
Found reference to 1de5c80b548
ValSerieItem: nptrs=1, skip=8
Found reference to 1de5c80b560
```

# Conclusion

We've implemented another missing piece of our GC: the ability to find the references in an object, essential to walk the reference graph. In the process, we've uncovered some of the subtleties of the objects layout in .NET. In the next article, we should finally be able to build a graph of reachable objects, with a few limitations.

As usual, the full code is available on [GitHub](https://github.com/kevingosse/ManagedDotnetGC/tree/Part5).


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
