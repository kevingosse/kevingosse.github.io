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
- csharp
- programming
- debugging
- windbg
author: Kevin Gosse
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

![][image_ref_MSpnWU1DR0pQekh2WW4yamw0a0J0U2hnLnBuZw==]

The full code [is available here](https://gist.github.com/kevingosse/65b310d285a201a5487c0a32a48130d7).


[image_ref_MSpnWU1DR0pQekh2WW4yamw0a0J0U2hnLnBuZw==]: data:image/png;base64,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