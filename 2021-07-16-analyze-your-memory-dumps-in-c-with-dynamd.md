---
url: https://medium.com/@kevingosse/analyze-your-memory-dumps-in-c-with-dynamd-8e4b110b9d3a
canonical_url: https://medium.com/@kevingosse/analyze-your-memory-dumps-in-c-with-dynamd-8e4b110b9d3a
title: Analyze your memory dumps in C# with DynaMD
subtitle: Browse memory structures from a memory dump in C#, just like you would with
  ClrMD, but in a more fluent way.
slug: analyze-your-memory-dumps-in-c-with-dynamd
description: ""
tags:
- csharp
- dotnet
- debugging
- programming
- software-development
author: Kevin Gosse
username: kevingosse
---

# Analyze your memory dumps in C# with DynaMD

Whenever you need to analyze complex structures in a .NET memory dump, the WinDbg scripting API quickly shows its limits. In those cases, you can instead use [the ClrMD library](https://github.com/microsoft/clrmd), that will give you everything you need to inspect the memory dump from C# code.

Not everything is perfect however, and sometimes I feel like the ClrMD syntax does not feel “natural” enough. To take one concrete example, for an investigation I had to retrieve the URLs of the pending HTTP requests in a memory dump. To do so, I needed to:

* Retrieve the `[ConnectionGroup](https://docs.microsoft.com/en-us/dotnet/framework/additional-apis/connectiongroup?WT.mc_id=DT-MVP-5003493)` associated to the given host

* Iterate on the connections (stored in the `[connectionGroup.m_ConnectionList](https://docs.microsoft.com/en-us/dotnet/framework/additional-apis/m_connectionlist?WT.mc_id=DT-MVP-5003493)` list)

* Find the ones with a pending request and print the URL

With ClrMD, that code would look like:

```
class Program
{
    static void Main(string[] args)
    {
        var target = DataTarget.LoadDump(@"dump.dmp");
        var runtime = target.ClrVersions[0].CreateRuntime();

        var heap = runtime.Heap;

        foreach (var obj in heap.EnumerateObjects())
        {
            if (obj.Type?.Name != "System.Net.ConnectionGroup")
            {
                continue;
            }

            var connectionGroup = obj;

            var servicePoint = connectionGroup.ReadObjectField("m_ServicePoint");

            if (servicePoint.ReadStringField("m_Host") != "contoso.com")
            {
                continue;
            }

            var connections = connectionGroup.ReadObjectField("m_ConnectionList");
            var array = connections.ReadObjectField("_items").AsArray();

            for (int i = 0; i < array.Length; i++)
            {
                var connection = array.GetObjectValue(i);

                if (connection.IsNull)
                {
                    continue;
                }

                var currentRequest = connection.ReadObjectField("m_CurrentRequest");

                if (!currentRequest.IsNull)
                {
                    var uri = currentRequest.ReadObjectField("_Uri").ReadStringField("m_String");
                    Console.WriteLine(uri);
                }
            }
        }
    }
}
```
> *[ClrMD.cs view raw](https://gist.githubusercontent.com/kevingosse/33a887d7745d4c10eae4f1e7652b995f/raw/e94cbf103efcf1679450b8548da7f6e45425d8ff/ClrMD.cs)*

I feel like those repeated `ReadObjectField` calls add a lot of noise, and accessing the array is not straightforward.

To make all of this easier, I wrote the [DynaMD](https://github.com/kevingosse/dynamd) library. This is a helper that runs on top of ClrMD, and leverages the C# `dynamic` keyword to expose the structures in a much more fluent way. When rewritten with DynaMD, the previous code becomes:

```
static void Main(string[] args)
{
    var target = DataTarget.LoadDump(@"dump.dmp");
    var runtime = target.ClrVersions[0].CreateRuntime();

    var heap = runtime.Heap;

    foreach (var connectionGroup in heap.GetProxies("System.Net.ConnectionGroup"))
    {
        if ((string)connectionGroup.m_ServicePoint.m_Host != "contoso.com")
        {
            continue;
        }

        foreach (var connection in connectionGroup.m_ConnectionList._items)
        {
            if (connection?.m_CurrentRequest == null)
            {
                continue;
            }

            Console.WriteLine((string)connection.m_CurrentRequest._Uri.m_String);
        }
    }
}

```
> *[DynaMD.cs view raw](https://gist.githubusercontent.com/kevingosse/798b9f4142e7dc2f8827f73979b4c586/raw/7d5cc627c680408cad9d930b7ccac84b068011d7/DynaMD.cs)*

Of course it’s always a matter of personal preference, but I find this version much easier to write and read.

So, what is DynaMD capable of?

# Introduction to DynaMD

The first step is of course to install [the nuget package](https://www.nuget.org/packages/DynaMD/). The versions 1.0.8+ use ClrMD 2.*, you can use a previous version if for some reason you need ClrMD 1.*.

Then, there are multiple ways to retrieve a dynamic proxy to an object:

```
// Retrieve an instance by its address
var proxy = heap.GetProxy(0x00001000);

// Iterate on all instances of a given type
foreach (var proxy in heap.GetProxies<string>()) { }

// or (if you don't reference the type)
foreach (var proxy in heap.GetProxies("System.String")) { }
```
> *[GetProxy.cs view raw](https://gist.githubusercontent.com/kevingosse/08cec6c26950084882eb415989759301/raw/8b43ad0d57e4fd159e0e5a17a825729768bc8e32/GetProxy.cs)*

`proxy` is `dynamic`, so you don’t get any autocompletion. But, knowing the structure of the object, you can access any field just like you would with a “real” object:

```
Console.WriteLine(proxy.Value);
Console.WriteLine((string)proxy.Child.Name);
Console.WriteLine(proxy.Description.Size.Width * proxy.Description.Size.Height);
```
> *[Fields.cs view raw](https://gist.githubusercontent.com/kevingosse/ae6ccb49c962ca80f04de00457332c37/raw/03784cbfe11267feb12b4301b3c754e21e7b5ed8/Fields.cs)*

Only fields are supported, but automatic properties are translated:

```
class SomeType
{
    private int _backingField;
    public int Field1 => _backingField;
    public int Field2 { get; }
}

var proxy = heap.GetProxies<SomeType>().First();

var value1 = proxy._backingField; // Calling proxy.Field1 is not supported
var value2 = proxy.Field2; // Automatically translated to <Field2>k__BackingField
```
> *[Properties.cs view raw](https://gist.githubusercontent.com/kevingosse/b67fe224960582a06e4207cb43b622f4/raw/8dd185932cb078093bef6368fd7eca37793d40f7/Properties.cs)*

At some point, you’ll probably need to materialize the value of a field. This is done automatically for primitive types:

```
class SomeType
{
    public int IntValue;
    public double DoubleValue;
}

var proxy = heap.GetProxies<SomeType>().First();

int intValue = proxy.IntValue; // Automatically converted to an int
double doubleValue = proxy.DoubleValue; // Automatically converted to a double
```
> *[Primitives.cs view raw](https://gist.githubusercontent.com/kevingosse/88ef0251174293c5558af04d42712fe7/raw/af6b87197f4ca43a5f7debfb69f52df6c524b6a9/Primitives.cs)*

DynaMD also supports explicit conversion to `string` or [blittable structs](https://docs.microsoft.com/en-us/dotnet/framework/interop/blittable-and-non-blittable-types?WT.mc_id=DT-MVP-5003493):

```
struct BlittableStruct
{
    public int Value;
}

class SomeType
{
    public BlittableStruct StructValue;
    public DateTime DateTimeValue;
    public string StringValue;
}

var proxy = heap.GetProxies<SomeType>().First();

BlittableStruct structValue = (BlittableStruct)proxy.StructValue;
DateTime dateTimeValue = (DateTime)proxy.DateTimeValue;
string stringValue = (string)proxy.stringValue;
```
> *[structs.cs view raw](https://gist.githubusercontent.com/kevingosse/e1ee1c4c51b1c4c9281ec61cc8325d1a/raw/92419679a58599e9ecd831598f15d8f6a8ff8f67/structs.cs)*

> If you don’t know what “blittable structs” are, you can think of them as “any object that is stored as a single chunk of consecutive memory”. In other words, a blittable struct in C# would be a struct that only contains primitive types and other blittable structs. No references.

When dealing with arrays, you can directly enumerate the contents, get the length, or use an indexer:

```
class SomeType
{
    public int[] ArrayValue;
}

var proxy = heap.GetProxies<SomeType>().First();

Console.WriteLine("Length: " + proxy.ArrayValue.Length);
Console.WriteLine("First element: " + proxy.ArrayValue[0]);

foreach (int value in proxy.ArrayValue)
{
    Console.WriteLine(value);
}
```
> *[arrays.cs view raw](https://gist.githubusercontent.com/kevingosse/52e63f2cc7f7d8b22d653bf923d2318c/raw/7eaf67b85067078ccc478c56941db61fd9e679e3/arrays.cs)*

To retrieve the address of a proxified object, explicitly cast it to `ulong`. Also, calling `.ToString()` on a proxy will return the address encoded in hexadecimal:

```
var proxy = heap.GetProxy(0x1000);
var address = (ulong)proxy;
Console.WriteLine("{0:x2}", address); // 0x1000
Console.WriteLine(proxy); // 0x1000
```
> *[helpers.cs view raw](https://gist.githubusercontent.com/kevingosse/ca09ef17e1ca82b1e64a7a6d63862c10/raw/2f2d01c0ef1498ee4399c95c2276b46ba3679554/helpers.cs)*

To retrieve the instance of ClrType, call `GetClrType()`:

```
ClrType type = proxy.GetClrType();
```
> *[GetClrType.cs view raw](https://gist.githubusercontent.com/kevingosse/3457e6f8063cf36af49f0d3b3b948eb4/raw/85ef47099777eb004926eda7815eadb93e6abf24/GetClrType.cs)*

That’s all the library is capable of doing for now, but this should be enough to cover most scenarios. I’m still working on an experimental branch that would allow to cast a proxy to any reference type, I hope I’ll be able to release something in the coming months.

Feel free to contribute or report bugs [on the GitHub repository](https://github.com/kevingosse/dynamd)!


