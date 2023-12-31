---
url: dumping-stack-objects-with-clrmd-c002dab4651b
canonical_url: https://medium.com/criteo-engineering/dumping-stack-objects-with-clrmd-c002dab4651b
title: Dumping stack objects with ClrMD
subtitle: How to implement the WinDbg DumpStackObjects command in ClrMD
summary: How to implement the WinDbg DumpStackObjects command in ClrMD.
date: 2019-12-18
tags:
- dotnet
- clrmd
- debugging
author: Kevin Gosse
thumbnailImage: /images/dumping-stack-objects-with-clrmd-c002dab4651b-1.webp
---

The SOS extension for WinDbg defines a command, `!DumpStackObjects` (or `!dso`), that allows to list all the references on the stack for a given thread. It's convenient to retrieve the value of arguments or local variables of methods in the callstack. Recently, for an investigation, I ran across the need to dump those values for a few thousand threads. Obviously this is not something you'd want to do by hand, so I checked what was possible to do with ClrMD.

ClrMD defines an `EnumerateStackObjects` method on the `ClrThread` object, which seems to do exactly what's needed. To make sure, we can write this small program and compare the output with the !DumpStackObjects command:

```csharp
using System;
using System.Linq;
using Microsoft.Diagnostics.Runtime;

namespace DumpStackObjects
{
    class Program
    {
        static void Main(string[] args)
        {
            var target = DataTarget.LoadCrashDump(@"E:\CoreConsoleApp1.dmp");
            var runtime = target.ClrVersions[0].CreateRuntime();

            var thread = runtime.Threads.First(t => t.ManagedThreadId == 1);

            foreach (var reference in thread.EnumerateStackObjects())
            {
                Console.WriteLine($"{reference.Address:x16} {reference.Object:x16} {reference.Type.Name}");
            }
        }
    }
}
```

Let's see what's going on here. `EnumerateStackObjects` returns a list of `ClrRoot` instances. Those instances represent references. The `Address` field contains the address of the reference **on the stack**, whereas the `Object` field contains the address of the object the reference is pointing to (the value you're likely interested in). If we compare the output of our program with WinDbg, it seems indeed to match:

{{<image classes="fancybox" src="/images/dumping-stack-objects-with-clrmd-c002dab4651b-1.webp" title="ClrMD on the left, WinDbg/SOS on the right">}}

That is… on Windows. For my investigation, I needed to do the same thing with a Linux coredump. ClrMD is compatible with Linux, so it shouldn't be a problem, or so I thought. But when I ran the same code on Linux, no stack objects were found.

{{<image classes="fancybox" src="/images/dumping-stack-objects-with-clrmd-c002dab4651b-2.webp" title="LLDB/SOS on the left, ClrMD on the right">}}

Digging into the `EnumerateStackObjects` implementation, it seems that it gets the address range of the stack by inspecting `ClrThread.StackBase` and `ClrThread.StackLimit`, then checks every value in that range to find references. In my case, `StackBase` and `StackLimit` both returned 0, so it makes sense that the algorithm would then fail to find anything. So how are `StackBase` and `StackLimit` implemented?

In ClrMD, `StackBase` and `StackLimit` are read from the [thread environment block (TEB)](https://en.wikipedia.org/wiki/Win32_Thread_Information_Block). The contents of the TEB are retrieved directly from the DAC using the `GetThreadData` method. If we look at [the implementation of that function in the CoreCLR repository](https://github.com/dotnet/coreclr/blob/d2a8b55d4c944438df1599c9a78262bc48f07266/src/debug/daccess/request.cpp#L752), the problem becomes quite obvious:

```c++
#ifndef FEATURE_PAL
    threadData->teb = TO_CDADDR(thread->m_pTEB);
#else
    threadData->teb = NULL;
#endif
```

`FEATURE_PAL` is defined on all other platforms than Windows. It means that when running .net core on Linux, the TEB information of the thread will be empty. It actually makes sense since TEB is a Windows concept, but then how is the SOS DumpStackObjects function working on Linux?

This time, the answer lies [in the dotnet/diagnostics repository](https://github.com/dotnet/diagnostics/blob/367aa5cd433e5e5a9e128aee5acb8196231ac769/src/SOS/Strike/strike.cpp#L616). Here is a stripped down version with only the interesting bits:

```c++
HRESULT DumpStackObjectsRaw(size_t nArg, __in_z LPSTR exprBottom, __in_z LPSTR exprTop, BOOL bVerify)
{
    size_t StackTop = 0;
    size_t StackBottom = 0;
    if (nArg==0)
    {
        ULONG64 StackOffset;
        g_ExtRegisters->GetStackOffset(&StackOffset);

        StackTop = TO_TADDR(StackOffset);
    }
    else
    {
        // Additional logic to handle the arguments
    }
    
#ifndef FEATURE_PAL
    // Read StackBottom from the TEB
#endif
    
    if (StackBottom == 0)
        StackBottom = StackTop + 0xFFFF;
        
    // Browse the stack and retrieve the references
}
```

The code starts by getting the current stack pointer from the registers by calling the `GetStackOffset` method, and assigns it to `StackTop`. Then, **only on Windows**, it reads the TEB to get the address of the beginning the stack. For other platforms, there's a fallback code where `StackBottom` is initialized to `StackTop + 0xFFFF` if no value has been assigned. This hack puzzles me quite a bit. 0xFFFF (64KB) is much smaller than the default size of the stack, and at the same time I don't think it offers any guarantee that we won't read memory outside of the stack (I searched for a while but couldn't find any evidence that a guard page is allocated above the stack).

However surprising it may look, our goal is just to mimic the DumpStackObject output, so can we implement the same logic in ClrMD?

For that, we need a way to retrieve the value of the RSP register, to get the stack offset. This is done by calling the `GetThreadContext` method with the id of the thread. It takes as parameter a byte array (or an `IntPtr`), in which is serialized the contents of a structure representing the registers. That structure is `AMD64Context`, `ArmContext`, or `Arm64Context` depending on the target architecture. In our case, we only care about AMD64:

```csharp
var bytes = new byte[sizeof(AMD64Context)];

if (!target.DataReader.GetThreadContext(thread.OSThreadId, 0, (uint)bytes.Length, bytes))
{
    Console.WriteLine("GetThreadContext failed");
    return;
}

var context = MemoryMarshal.Read<AMD64Context>(bytes);
```

Once we have this information, we can start browsing the stack and search for references (the code is shamelessly copied [from the ClrMD samples](https://github.com/microsoft/dotnet-samples/blob/master/Microsoft.Diagnostics.Runtime/CLRMD/ClrStack/Program.cs#L81)):

```csharp
var stackTop = context.Rsp;
var stackLimit = stackTop + 0xFFFF;

for (ulong ptr = stackTop; ptr <= stackLimit; ptr += (ulong)runtime.PointerSize)
{
    ulong obj;

    if (!runtime.ReadPointer(ptr, out obj))
    {
        break;
    }       

    var type = heap.GetObjectType(obj);

    if (type == null || type.IsFree)
    {
        continue;
    }

    Console.WriteLine($"{ptr:x16} {obj:x16} {type.Name}");
}
```

We start by trying to dereference any value we find in the stack, then we call `ClrHeap.GetObjectType` to find if it points to a valid object. If so, we display the result to the console.

I tried running this code and it worked fine for a few hundred threads, then it crashed with a segmentation fault. I managed to track it down to a call to `ClrHeap.GetObjectType` with a specific address. It's likely that we found a pointer to a value outside of the managed heap, which is causing some unpredictable behavior. How is SOS coping with this? Looking back at [the implementation](https://github.com/dotnet/diagnostics/blob/master/src/SOS/Strike/util.cpp#L1988), it looks like we forgot a sanity check:

```c++
// rule out pointers that are outside of the gc heap.
if (g_snapshot.GetHeap(objAddr) == NULL)
    return;
```

With ClrMD, we can achieve the same result by enumerating the GC segments and making sure our address is inside one of those:

```csharp
private static bool IsInHeap(ClrHeap heap, ulong obj)
{
    foreach (var segment in heap.Segments)
    {
        if (segment.Start <= obj && segment.End > obj)
        {
            return true;
        }
    }

    return false;
}
```

Now we have everything we need to finish our implementation of DumpStackObjects for Linux:

```csharp
using System;
using System.Linq;
using System.Runtime.InteropServices;
using Microsoft.Diagnostics.Runtime;

namespace DumpStackObjects
{
    class Program
    {
        static unsafe void Main(string[] args)
        {
            var threadId = int.Parse(args[1]);

            var target = DataTarget.LoadCoreDump(args[0]);
            var runtime = target.ClrVersions[0].CreateRuntime();
            var heap = runtime.Heap;

            var thread = runtime.Threads.First(t => t.ManagedThreadId == threadId);

            var bytes = new byte[sizeof(AMD64Context)];

            if (!target.DataReader.GetThreadContext(thread.OSThreadId, 0, (uint)bytes.Length, bytes))
            {
                Console.WriteLine("GetThreadContext failed");
                return;
            }

            var context = MemoryMarshal.Read<AMD64Context>(bytes);

            var stackTop = context.Rsp;
            var stackLimit = stackTop + 0xFFFF;

            for (ulong ptr = stackTop; ptr <= stackLimit; ptr += (ulong)runtime.PointerSize)
            {
                ulong obj;

                if (!runtime.ReadPointer(ptr, out obj))
                {
                    break;
                }

                if (!IsInHeap(heap, obj))
                {
                    continue;
                }

                var type = heap.GetObjectType(obj);
                
                if (type == null || type.IsFree)
                {
                    continue;
                }

                Console.WriteLine($"{ptr:x16} {obj:x16} {type.Name}");
            }
        }

        private static bool IsInHeap(ClrHeap heap, ulong obj)
        {
            foreach (var segment in heap.Segments)
            {
                if (segment.Start <= obj && segment.End > obj)
                {
                    return true;
                }
            }

            return false;
        }
    }
}
```

And this time, we get the same output as the `!DumpStackObjects` command, as expected:

{{<image classes="fancybox" src="/images/dumping-stack-objects-with-clrmd-c002dab4651b-3.webp" >}}