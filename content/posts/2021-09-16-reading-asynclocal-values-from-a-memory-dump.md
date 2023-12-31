---
url: reading-asynclocal-values-from-a-memory-dump-8be356a46b29
canonical_url: https://medium.com/@kevingosse/reading-asynclocal-values-from-a-memory-dump-8be356a46b29
title: Reading AsyncLocal values from a memory dump
subtitle: This article explains how AsyncLocal values are stored in .NET and how to
  retrieve them from a memory dump.
date: 2021-09-16
description: ""
tags:
- dotnet
- debugging
- windbg
- clrmd
- async
author: Kevin Gosse
thumbnailImage: /images/reading-asynclocal-values-from-a-memory-dump-8be356a46b29-1.webp
---

This article explains how [`AsyncLocal`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.asynclocal-1?view=net-5.0&WT.mc_id=DT-MVP-5003493) values are stored in .NET and how to retrieve them from a memory dump. Note that the code provided is written for .NET 5, but should be able to work for .NET Framework with minor modifications (the name of some internal fields have changed across versions).

# Preparing the memory dump

First thing first, we need to prepare the memory dump that will serve as example for the whole article. For that, I used the following code:

```csharp
using System;
using System.Diagnostics;
using System.Threading;

namespace ClientApp
{
    internal class Program
    {
        private const int Total = 17;

        private readonly AsyncLocal<int>[] _asyncLocals;

        public Program()
        {
            _asyncLocals = new AsyncLocal<int>[Total];

            for (int i = 0; i < _asyncLocals.Length; i++)
            {
                _asyncLocals[i] = new AsyncLocal<int>();
            }
        }

        static void Main(string[] args)
        {
            var program = new Program();

            var countdown = new CountdownEvent(Total);

            for (int i = 0; i < Total; i++)
            {
                var thread = new Thread(state =>
                {
                    var index = (int)state;

                    Thread.CurrentThread.Name = index.ToString();

                    for (int j = 0; j <= index; j++)
                    {
                        program._asyncLocals[j].Value = j;
                    }

                    countdown.Signal();

                    Thread.Sleep(Timeout.Infinite);
                });

                thread.Start(i);
            }

            countdown.Wait();

            Console.WriteLine("Ready to capture dump. Process id {0}", Process.GetCurrentProcess().Id);
            Console.ReadLine();
        }
    }
}
```

The program creates an array of `AsyncLocal<int>` of 17 elements. 17 is not a random number, its significance will become clear in a moment. Then the program creates a few threads, and have them store a different number of asynclocal values, such as thread 1 will store 1 asynclocal value, thread 2 will store 2, and so on until thread 17. When all the values are initialized, the program displays a prompt and pauses to give time to capture a memory dump. This can be done using [the procdump tool](https://docs.microsoft.com/en-us/sysinternals/downloads/procdump?WT.mc_id=DT-MVP-5003493):

```bash
$ procdump -ma <pid>
```

# Inspecting the memory dump with WinDbg

Now that we have our memory dump, let's start by inspecting it with WinDbg. We'll start by retrieving the array of `AsyncLocal<int>`. Recent versions of WinDbg automatically load the SOS extension when dealing with .NET Core memory dumps, so that's one less thing to do. We can use the `dumpheap` command to locate our instance of `Program`, and from there access the array.

For that, we first use `dumpheap -stat -type ClientApp.Program`to get the address of the MT, and then we feed it to `dumpheap -mt` to get the address of the instance:

```
0:000> !dumpheap -stat -type ClientApp.Program
Statistics:
              MT    Count    TotalSize Class Name
00007ffb8e8e4968        1           24 ClientApp.Program
00007ffb8e9141d0        1           40 ClientApp.Program+<>c__DisplayClass3_0
Total 2 objects

0:000> !dumpheap -mt 00007ffb8e8e4968 
         Address               MT     Size
000001c53436bb08 00007ffb8e8e4968       24     

Statistics:
              MT    Count    TotalSize Class Name
00007ffb8e8e4968        1           24 ClientApp.Program
Total 1 objects
```

Then we use `dumpobj` to dump the instance and from there we get the address of the value stored in the `_asyncLocals` field:

```
0:000> !dumpobj 000001c53436bb08
Name:        ClientApp.Program
MethodTable: 00007ffb8e8e4968
EEClass:     00007ffb8e8fa988
Size:        24(0x18) bytes
File:        C:\Users\kevin.gosse\source\repos\ClrmdAsyncLocal\ClientApp\bin\Release\net5.0\ClientApp.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007ffb8e91b5c0  4000004        8 ...ivate.CoreLib]][]  0 instance 000001c53436bb78 _asyncLocals

0:000> !dumpobj 000001c53436bb78
Name:        System.Threading.AsyncLocal`1[[System.Int32, System.Private.CoreLib]][]
MethodTable: 00007ffb8e91b5c0
EEClass:     00007ffb8e8067c8
Size:        160(0xa0) bytes
Array:       Rank 1, Number of elements 17, Type CLASS (Print Array)
Fields:
None
```

Now we can dump the contents of the array using `dumparray` and inspect the individual items:

```
0:000> !dumparray 000001c53436bb78
Name:        System.Threading.AsyncLocal`1[[System.Int32, System.Private.CoreLib]][]
MethodTable: 00007ffb8e91b5c0
EEClass:     00007ffb8e8067c8
Size:        160(0xa0) bytes
Array:       Rank 1, Number of elements 17, Type CLASS
Element Methodtable: 00007ffb8e91b4d0
[0] 000001c53436bc18
[1] 000001c53436bc30
[2] 000001c53436bc48
[3] 000001c53436bc60
[4] 000001c53436bc78
[5] 000001c53436bc90
[6] 000001c53436bca8
[7] 000001c53436bcc0
[8] 000001c53436bcd8
[9] 000001c53436bcf0
[10] 000001c53436bd08
[11] 000001c53436bd20
[12] 000001c53436bd38
[13] 000001c53436bd50
[14] 000001c53436bd68
[15] 000001c53436bd80
[16] 000001c53436bd98

0:000> !dumpobj 000001c53436bc18
Name:        System.Threading.AsyncLocal`1[[System.Int32, System.Private.CoreLib]]
MethodTable: 00007ffb8e91b4d0
EEClass:     00007ffb8e934ce8
Size:        24(0x18) bytes
File:        C:\Program Files\dotnet\shared\Microsoft.NETCore.App\5.0.10\System.Private.CoreLib.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007ffb8e940048  4000a28        8 ...Private.CoreLib]]  0 instance 0000000000000000 m_valueChangedHandler
```

Immediately we can see what the issue is going to be: in our instance of `AsyncLocal` there is… nothing. Literally nothing at all is stored inside, unless you provided a `valueChanged` handler (in which case, you will only find the address of your handler).

So where are those values stored? On the `Thread` object itself. The instance of `AsyncLocal` is just used as a key to retrieve the value from the execution context. If we take one thread at random, we can find all the asynclocal values from that thread stored in the `m_LocalValues` field of the execution context:

```
0:000> !DumpObj /d 000001c53436be40
Name:        System.Threading.Thread
MethodTable: 00007ffb8e8ca8c0
EEClass:     00007ffb8e8b67d0
Size:        72(0x48) bytes
File:        C:\Program Files\dotnet\shared\Microsoft.NETCore.App\5.0.10\System.Private.CoreLib.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007ffb8e91c1e8  40009e6        8 ....ExecutionContext  0 instance 000001c53436dea0 _executionContext
0000000000000000  40009e7       10 ...ronizationContext  0 instance 0000000000000000 _synchronizationContext
00007ffb8e8c7a90  40009e8       18        System.String  0 instance 000001c53436d4f0 _name
00007ffb8e8c1a30  40009e9       20      System.Delegate  0 instance 0000000000000000 _delegate
00007ffb8e800c68  40009ea       28        System.Object  0 instance 0000000000000000 _threadStartArg
00007ffb8e80ee60  40009eb       30        System.IntPtr  1 instance 000001C534165830 _DONT_USE_InternalThread
00007ffb8e80b258  40009ec       38         System.Int32  1 instance                2 _priority
00007ffb8e80b258  40009ed       3c         System.Int32  1 instance                4 _managedThreadId
00007ffb8e80b258  40009ef      914         System.Int32  1   static                7 s_optimalMaxSpinWaitsPerSpinIteration
00007ffb8e807238  40009f0      918       System.Boolean  1   static                0 s_isProcessorNumberReallyFast
0000000000000000  40009f1      760                       0   static 0000000000000000 s_asyncLocalPrincipal
00007ffb8e8ca8c0  40009f2       18 ....Threading.Thread  0 TLstatic  t_currentThread
    >> Thread:Value 359c:000001c53436bf88 1940:000001c53436be40 7214:000001c53436c050 7034:000001c53436c128 6e38:000001c53436c200 7220:000001c53436c2d8 5684:000001c53436c3b0 4364:000001c53436c488 63ec:000001c53436c560 1970:000001c53436c638 6bcc:000001c53436c710 6890:000001c53436c7e8 69d0:000001c53436c8c0 1dc8:000001c53436c998 1db4:000001c53436ca70 6aa8:000001c53436cb48 6b94:000001c53436cc20 628:000001c53436ccf8 <<

0:000> !DumpObj /d 000001c53436dea0
Name:        System.Threading.ExecutionContext
MethodTable: 00007ffb8e91c1e8
EEClass:     00007ffb8e935090
Size:        40(0x28) bytes
File:        C:\Program Files\dotnet\shared\Microsoft.NETCore.App\5.0.10\System.Private.CoreLib.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007ffb8e91e7c8  4000a69        8 ...syncLocalValueMap  0 instance 000001c53436de80 m_localValues
00007ffb8e91eb78  4000a6a       10 ...ing.IAsyncLocal[]  0 instance 0000000000000000 m_localChangeNotifications
00007ffb8e807238  4000a6b       18       System.Boolean  1 instance                0 m_isFlowSuppressed
00007ffb8e807238  4000a6c       19       System.Boolean  1 instance                0 m_isDefault
00007ffb8e91c1e8  4000a67      7f8 ....ExecutionContext  0   static 000001c53436bfd0 Default
00007ffb8e91c1e8  4000a68      800 ....ExecutionContext  0   static 000001c53436c010 DefaultFlowSuppressed

0:000> !DumpObj /d 000001c53436de80
Name:        System.Threading.AsyncLocalValueMap+OneElementAsyncLocalValueMap
MethodTable: 00007ffb8e9408b8
EEClass:     00007ffb8e935de8
Size:        32(0x20) bytes
File:        C:\Program Files\dotnet\shared\Microsoft.NETCore.App\5.0.10\System.Private.CoreLib.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007ffb8e91b248  4000a2d        8 ...ading.IAsyncLocal  0 instance 000001c53436bc18 _key1
00007ffb8e800c68  4000a2e       10        System.Object  0 instance 000001c53436de68 _value1
```

The `_key1` field contains the address of the `AsyncLocal` instance, and the `_value1` field contains the value. So to check all the values stores in an `AsyncLocal` instance, we need to inspect every single thread. Not only that, but the way the values are stored is going to change depending on how many different `AsyncLocal` values are associated to a given thread. Managing that in WinDbg is going to be very tricky, so it's time to switch to ClrMD.

# Inspecting the memory dump with ClrMD

Our ClrMD program will open the memory dump, find the `AsyncLocal` array, and extract all the values for every thread. To make things easier, we're going to use [the DynaMD library](/analyze-your-memory-dumps-in-c-with-dynamd-8e4b110b9d3a).

First thing first, we open the memory dump, and we locate the first (and only) instance of `AsyncLocal<int>[]` thanks to the `GetProxies` extension method. Then we iterate on every instance and give them to a helper method that will extract a list of thread ids and their associated values:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using Microsoft.Diagnostics.Runtime;

namespace ClrmdAsyncLocal
{
    internal class Program
    {
        static void Main(string[] args)
        {
            const string path = @"C:\Users\kevin.gosse\Desktop\blog\asynclocal\ClientApp.exe_210916_114422.dmp";

            var target = DataTarget.LoadDump(path);
            var runtime = target.ClrVersions.First().CreateRuntime();
            var heap = runtime.Heap;

            var array = heap.GetProxies<AsyncLocal<int>[]>().First();

            foreach (var item in array)
            {
                Console.WriteLine($"Values stored in AsyncLocal {item}:");

                foreach (var (threadId, value) in ExtractAsyncLocalValues(heap, (ulong)item))
                {
                    Console.WriteLine($" - {threadId}: {(int)value}");
                }
            }

            Console.ReadLine();
        }

        private static IEnumerable<(int threadId, dynamic value)> ExtractAsyncLocalValues(ClrHeap heap, ulong address)
        {
            // TODO
        }
    }
}
```

The helper method will iterate on every thread, inspect the `AsyncLocal` values stored in the execution context, and filter them to only return the one associated to the given instance:

```csharp
private static IEnumerable<(int threadId, dynamic value)> ExtractAsyncLocalValues(ClrHeap heap, ulong address)
{
    foreach (var thread in heap.GetProxies<Thread>())
    {
        int threadId = thread._managedThreadId;
        var localValues = thread._executionContext?.m_localValues;

        foreach ((dynamic key, dynamic value) pair in ReadAsyncLocalStorage(localValues))
        {
            if ((ulong)pair.key == address)
            {
                yield return (threadId, pair.value);
                break;
            }
        }
    }
}

private static IEnumerable<(dynamic key, dynamic value)> ReadAsyncLocalStorage(dynamic storage)
{
    // TODO        
}
```

Now we just need to extract the values from the `AsyncLocal` storage… Which is where things get fancy. It turns out there's not one but many different storage implementations, depending on how many values the current thread is storing.

### From 1 to 3 values

If a given thread stores one to three different `AsyncLocal` values, it will use an instance of `OneElementAsyncLocalValueMap`, `TwoElementAsyncLocalValueMap`, or… `ThreeElementAsyncLocalValueMap`. Those types just have a list of fields to store the keys and the values, `_key1` / `_key2` / `_key3` and `_value1` / `_value2` / `_value3` respectively. Extracting the values is just a matter of reading those fields:

```csharp
private static IEnumerable<(dynamic key, dynamic value)> ReadOneElementAsyncLocalValueMap(dynamic storage)
{
    yield return (storage._key1, storage._value1);
}

private static IEnumerable<(dynamic key, dynamic value)> ReadTwoElementAsyncLocalValueMap(dynamic storage)
{
    yield return (storage._key1, storage._value1);
    yield return (storage._key2, storage._value2);
}

private static IEnumerable<(dynamic key, dynamic value)> ReadThreeElementAsyncLocalValueMap(dynamic storage)
{
    yield return (storage._key1, storage._value1);
    yield return (storage._key2, storage._value2);
    yield return (storage._key3, storage._value3);
}
```

### From 4 to 16 values

If a given thread stores four to sixteen values, it will thankfully stop declaring a field for every value and instead use an array-backed storage: `MultiElementAsyncLocalValueMap`. All the keys and values are stored in the `_keyValues` field as an array of `KeyValuePair<IAsyncLocal, object>`. The array is allocated with exactly the right size, so we just need to iterate on it to return the contents. It's very straightforward with DynaMD:

```csharp
private static IEnumerable<(dynamic key, dynamic value)> ReadMultiElementAsyncLocalValueMap(dynamic storage)
{
    foreach (var kvp in storage._keyValues)
    {
        yield return (kvp.key, kvp.value);
    }
}
```

### More than 16 values

To read from an `AsyncLocal` storage, the runtime has to find the value associated to a given key. Iterating on an array becomes less and less efficient as the number of values increases, so the runtime fallbacks on a `ManyElementAsyncLocalValueMap` when that number gets above 16. This type directly inherits from `Dictionary<IAsyncLocal, object>`, so to extract the values we need to iterate on the `_entries` field:

```csharp
private static IEnumerable<(dynamic key, dynamic value)> ReadManyElementAsyncLocalValueMap(dynamic storage)
{
    foreach (var entry in storage._entries)
    {
        if (entry.key != null)
        {
            yield return (entry.key, entry.value);
        }
    }
}
```

### Putting everything together

At this point, you may be wondering how everything fits together. Every time you add or remove a value from an `AsyncLocal` instance, the runtime will allocate a new storage based on the new number of elements, and copy everything from the old to the new storage. This makes adding/removing values **very** inefficient, so use `AsyncLocal` with care.

In any case, we now have helper methods for every possible `AsyncLocal` storage, so we can just call the appropriate one in our `ReadAsyncLocalStorage` method:

```csharp
private static IEnumerable<(dynamic key, dynamic value)> ReadAsyncLocalStorage(dynamic storage)
{
    if (storage == null)
    {
        return Enumerable.Empty<(dynamic key, dynamic value)>();
    }

    ClrType type = storage.GetClrType();

    return type.Name switch
    {
        "System.Threading.AsyncLocalValueMap+OneElementAsyncLocalValueMap" => ReadOneElementAsyncLocalValueMap(storage),
        "System.Threading.AsyncLocalValueMap+TwoElementAsyncLocalValueMap" => ReadTwoElementAsyncLocalValueMap(storage),
        "System.Threading.AsyncLocalValueMap+ThreeElementAsyncLocalValueMap" => ReadThreeElementAsyncLocalValueMap(storage),
        "System.Threading.AsyncLocalValueMap+MultiElementAsyncLocalValueMap" => ReadMultiElementAsyncLocalValueMap(storage),
        "System.Threading.AsyncLocalValueMap+ManyElementAsyncLocalValueMap" => ReadManyElementAsyncLocalValueMap(storage),
        _ => throw new InvalidOperationException($"Unexpected asynclocal storage type: {type.Name}")
    };
}
```

And that's it! Now we can run the program and inspect the values for every `AsyncLocal` instance.

{{<image classes="fancybox center" src="/images/reading-asynclocal-values-from-a-memory-dump-8be356a46b29-1.webp" >}}

The full code [is available here](https://gist.github.com/kevingosse/65b310d285a201a5487c0a32a48130d7).
