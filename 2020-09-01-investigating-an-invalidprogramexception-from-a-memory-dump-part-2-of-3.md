---
url: https://medium.com/@kevingosse/investigating-an-invalidprogramexception-from-a-memory-dump-part-2-of-3-daaecd8f3cf4
canonical_url: https://medium.com/@kevingosse/investigating-an-invalidprogramexception-from-a-memory-dump-part-2-of-3-daaecd8f3cf4
title: Investigating an InvalidProgramException from a memory dump (part 2 of 3)
subtitle: Part 2 of the investigation, where we extract the dynamic IL from the memory
  dump
slug: investigating-an-invalidprogramexception-from-a-memory-dump-part-2-of-3
description: ""
tags:
- csharp
- dotnet
- debugging
- programming
- lldb
author: Kevin Gosse
username: kevingosse
---

# Investigating an InvalidProgramException from a memory dump (part 2 of 3)

In this series of article, we’re retracing how I debugged an `InvalidProgramException`, caused by a bug in the Datadog profiler, from a memory dump sent by a customer.

* [Part 1: Preliminary exploration](https://medium.com/@kevingosse/investigating-an-invalidprogramexception-from-a-memory-dump-part-1-of-3-bce634460cc3)

* Part 2: Finding the generated IL

* [Part 3: Identifying the error and fixing the bug](https://medium.com/@kevingosse/investigating-an-invalidprogramexception-from-a-memory-dump-part-3-of-3-c1d912075cb1)

Let’s start with a quick reminder. The profiler works by rewriting the IL of interesting methods to inject instrumentation code. The `InvalidProgramException` is thrown by the JIT when trying to compile the IL emitted by the profiler, which must be somehow invalid. [The first part](https://medium.com/@kevingosse/investigating-an-invalidprogramexception-from-a-memory-dump-part-1-of-3-bce634460cc3) was about identifying in what method the exception was thrown, and I ended up concluding that `Npgsql.PostgresDatabaseInfo.LoadBackendTypes` was the culprit. The second part is going to be about how to find the generated IL for that method.

# Finding the generated IL

`Npgsql.PostgresDatabaseInfo.LoadBackendTypes` is an asynchronous method. The logic of an async method is stored in the `MoveNext` method of its state-machine, so that’s the one I was interested in.

There’s a function in `dotnet-dump` to display the IL of a method: `dumpil`. This method requires the MethodDescriptor (MD) of the target method, so I needed to find it for `MoveNext`.

I started by dumping all the types in the module, using the `dumpmodule -mt` command, to find the state-machine:

```
```
> dumpmodule -mt 00007FCF13EC67D8

Name: /Npgsql.dll
Attributes:              PEFile SupportsUpdateableMethods IsFileLayout
Assembly:                0000000002a3f590
BaseAddress:             00007FCF8079E000
PEFile:                  0000000002A638B0
ModuleId:                00007FCF13EC7FE0
ModuleIndex:             0000000000000091
LoaderHeap:              0000000000000000
TypeDefToMethodTableMap: 00007FCF13F30020
TypeRefToMethodTableMap: 00007FCF13F31270
MethodDefToDescMap:      00007FCF13F31EE8
FieldDefToDescMap:       00007FCF13F392E0
MemberRefToDescMap:      0000000000000000
FileReferencesMap:       00007FCF13F3F740
AssemblyReferencesMap:   00007FCF13F3F748
MetaData start address:  00007FCF807DF410 (496736 bytes)

Types defined in this module

              MT          TypeDef Name
------------------------------------------------------------------------------
00007fcf161d7698 0x02000002 <>f__AnonymousType0`2
00007fcf161d7bb0 0x02000003 <>f__AnonymousType1`2
[...]
00007fcf16509c10 0x020001b0 Npgsql.PostgresDatabaseInfo+<LoadBackendTypes>d__16
[...]
```
> *[dumpmodule -mt.md view raw](https://gist.githubusercontent.com/kevingosse/5eabd02f0418fb318247789cdc0512cf/raw/9506e3159518c3b293627e14961680d077abd742/dumpmodule%20-mt.md)*

This gave me the MT (MethodTable) of the state-machine type: `00007fcf16509c10`. Then I fed it to the `dumpmt -md` command to get the MD:

```
```
> dumpmt -md 00007fcf16509c10

EEClass:         00007FCF164EBD70
Module:          00007FCF13EC67D8
Name:            Npgsql.PostgresDatabaseInfo+<LoadBackendTypes>d__16
mdToken:         00000000020001B0
File:            /Npgsql.dll
BaseSize:        0x60
ComponentSize:   0x0
DynamicStatics:  false
ContainsPointers true
Slots in VTable: 8
Number of IFaces in IFaceMap: 1
--------------------------------------
MethodDesc Table
           Entry       MethodDesc    JIT Name
[...]
00007FCF16265300 00007FCF16509B58   NONE Npgsql.PostgresDatabaseInfo+<LoadBackendTypes>d__16.MoveNext()
[...]
```
```
> *[dumpmt -md.md view raw](https://gist.githubusercontent.com/kevingosse/20b1221bb28df7b0d31dcd8ac9a0d237/raw/c9210bbbf5630176aca02f955dd4bac08d86b123/dumpmt%20-md.md)*

The command outputs all the methods from the given type, and from there we can see that our MD is `00007FCF16509B58`.

Unfortunately, the `dumpil` command returned the original IL, not the rewritten one. Looking for ideas, I used the `dumpmd` command to get more information about the method:

```
```
> dumpmd 00007FCF16509B58

Method Name:          Npgsql.PostgresDatabaseInfo+<LoadBackendTypes>d__16.MoveNext()
Class:                00007fcf164ebd70
MethodTable:          00007fcf16509c10
mdToken:              0000000006000D24
Module:               00007fcf13ec67d8
IsJitted:             no
Current CodeAddr:     ffffffffffffffff
Version History:
  ILCodeVersion:      0000000000000000
  ReJIT ID:           0
  IL Addr:            0000000000000000
     CodeAddr:           0000000000000000  (OptimizedTier1)
     NativeCodeVersion:  00007FCE84156100
     CodeAddr:           0000000000000000  (QuickJitted)
     NativeCodeVersion:  0000000000000000
```
```
> *[dumpmd.md view raw](https://gist.githubusercontent.com/kevingosse/e81fb9ee99a5f790567a4610857fc32e/raw/13ee703cb13b6691f795bbb21e9c8b9bae0eb7cc/dumpmd.md)*

Interestingly, the method was marked as not jitted. In hindsight, that makes sense. We rewrite the method using the `[JitCompilationStarted](https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilercallback-jitcompilationstarted-method?WT.mc_id=DT-MVP-5003493)` event of the profiler API. The JIT then tries to compile it, fails, and throws away the rewritten IL.

> *Fun fact for those who know about [tiered compilation](https://github.com/dotnet/coreclr/blob/master/Documentation/design-docs/tiered-compilation.md): you may have noticed in the `dumpmd` output that there are two versions of the method, QuickJitted and OptimizedTier1, despite the `IsJitted` flag being false. I managed to reproduce this on a test app with a profiler emitting bad IL: after calling the method 30 times, the tiered JIT promotes it to tier 1, even though the method was never jitted successfully*

Dead-end? I really didn’t want to give up after going through the tedious process of finding the method, so I decided to get creative. The same way that I managed to find the `InvalidProgramException` on the heap even though it wasn’t referenced anymore, I figured out that there could still be traces of the generated IL somewhere.

To give the rewritten IL to the JIT, the profiler uses the `[SetILFunctionBody](https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilerinfo-setilfunctionbody-method?WT.mc_id=DT-MVP-5003493)` API. What’s interesting about it is that the buffer, used to write the IL, is provided by the JIT own’s allocator. Quoting the documentation:

```
Use the ICorProfilerInfo::GetILFunctionBodyAllocator method to allocate space for the new method to ensure that the buffer is compatible.
```

Maybe I could find traces of the IL in whatever data structure is used internally by the body allocator? Unfortunately, the allocator [is just a call to the `new` operator](https://github.com/dotnet/runtime/blob/v5.0.0-preview.8.20407.11/src/coreclr/src/vm/proftoeeinterfaceimpl.cpp#L463):

```
//---------------------------------------------------------------------------------------
// Profiler entrypoint to allocate space from this module's heap.
//
// Arguments
//      cb - size in bytes of allocation request
//
// Return value
//      pointer to allocated memory, or NULL if there was an error

void * STDMETHODCALLTYPE ModuleILHeap::Alloc(ULONG cb)
{
    CONTRACTL
    {
        // Yay!
        NOTHROW;

        // (see GC_TRIGGERS comment below)
        CAN_TAKE_LOCK;

        // Allocations using loader heaps below enter a critsec, which switches
        // to preemptive, which is effectively a GC trigger
        GC_TRIGGERS;

        // Yay!
        MODE_ANY;

    }
    CONTRACTL_END;

    LOG((LF_CORPROF, LL_INFO1000, "**PROF: ModuleILHeap::Alloc 0x%08xp.\n", cb));

    if (cb == 0)
    {
        return NULL;
    }

    return new (nothrow) BYTE[cb];
}
```
> *[proftoeeinterfaceimpl.cpp view raw](https://gist.githubusercontent.com/kevingosse/ca7497bd3331a93a7df6fe5f0205e965/raw/de5487c268fd771223efcb69dcbb5d37c9b35a7f/proftoeeinterfaceimpl.cpp)*

I have no clue how the `new` operator works in C++, so I decided to follow another lead. What happens to that buffer after it’s given to the `SetILFunctionBody` method? I’m not going to show [the method implementation](https://github.com/dotnet/runtime/blob/v5.0.0-preview.8.20407.11/src/coreclr/src/vm/proftoeeinterfaceimpl.cpp#L4428), but the interesting bit is how it ends up calling `Module::SetDynamicIL`. In turn, `SetDynamicIL` stores the IL body in an internal table (this time I’m showing the implementation because it’ll be important for later):

```
void Module::SetDynamicIL(mdToken token, TADDR blobAddress, BOOL fTemporaryOverride)
{
    DynamicILBlobEntry entry = {mdToken(token), TADDR(blobAddress)};

    // Lazily allocate a Crst to serialize update access to the info structure.
    // Carefully synchronize to ensure we don't leak a Crst in race conditions.
    if (m_debuggerSpecificData.m_pDynamicILCrst == NULL)
    {
        InitializeDynamicILCrst();
    }

    CrstHolder ch(m_debuggerSpecificData.m_pDynamicILCrst);

    // Figure out which table to fill in
    PTR_DynamicILBlobTable &table(fTemporaryOverride ? m_debuggerSpecificData.m_pTemporaryILBlobTable
                                                     : m_debuggerSpecificData.m_pDynamicILBlobTable);

    // Lazily allocate the hash table.
    if (table == NULL)
    {
        table = PTR_DynamicILBlobTable(new DynamicILBlobTable);
    }
    table->AddOrReplace(entry);
}
```
> *[ceeload.cpp view raw](https://gist.githubusercontent.com/kevingosse/8c2d5b6301930d9dff9e613f7c980ec7/raw/5fa0c4229f0b247132f373a16b9709fe9e4050e1/ceeload.cpp)*

`fTemporaryOverride` is false in this codepath, so `m_debuggerSpecificData.m_pDynamicILBlobTable` is used to store the IL. If I could find the address of that table in the memory dump, then maybe I could retrieve the generated IL!

[As I shown in a previous article](https://medium.com/@kevingosse/check-what-net-core-gc-keywords-are-enabled-without-a-debugger-d616745c0d0e), it’s possible to export all the symbols of a module on Linux by using the `nm` command. So I tried looking for `m_debuggerSpecificData`, but no luck:

```
> nm -C libcoreclr.so | grep m_debuggerSpecificData
>
```

How could I possibly find this structure without symbols?

I firmly believe that [debugging is a creative process](https://www.youtube.com/watch?v=uXOl5MXglWs). So I took a step back and started thinking. When `Module::SetDynamicIL` is called, the runtime is capable, somehow, of locating that structure. So the answer, whatever it is, must be somewhere in the assembly code of that method.

> *Reading this article makes it sound like it’s an instantaneous process, but locating `m_debuggerSpecificData` without symbols is the result of 2 hours of trial and error and bouncing ideas back and forth with my former coworkers [Christophe Nasarre](https://twitter.com/chnasarre) and [Gregory Leocadie](https://twitter.com/GregoryLeocadie)
In the process, I also discovered that [a ISOSDacInterface7 is being implemented for .NET 5](https://github.com/dotnet/coreclr/pull/26227), and it has [all the facilities I needed to find the dynamic IL](https://github.com/dotnet/runtime/blob/v5.0.0-preview.8.20407.11/src/coreclr/src/debug/daccess/request.cpp#L4425). *sigh**

Fortunately, that method is exported in the symbols:

```
> nm -C libcoreclr.so | grep Module::SetDynamicIL
0000000000543da0 t Module::SetDynamicIL(unsigned int, unsigned long, int)
```

I used gdb to decompile it:

```
> gdb ./libcoreclr.so -batch -ex "set disassembly-flavor intel" -ex "disassemble Module::SetDynamicIL"

Dump of assembler code for function Module::SetDynamicIL(unsigned int, unsigned long, int):
   0x26bf00 <+0>:     push   rbp
   0x26bf01 <+1>:     mov    rbp,rsp
   0x26bf04 <+4>:     push   r15
   0x26bf06 <+6>:     push   r14
   0x26bf08 <+8>:     push   rbx
   0x26bf09 <+9>:     sub    rsp,0x18
   0x26bf0d <+13>:    mov    r15d,ecx
   0x26bf10 <+16>:    mov    rbx,rdi
   0x26bf13 <+19>:    mov    rax,QWORD PTR fs:0x28
   0x26bf1c <+28>:    mov    QWORD PTR [rbp-0x20],rax
   0x26bf20 <+32>:    mov    DWORD PTR [rbp-0x30],esi
   0x26bf23 <+35>:    mov    QWORD PTR [rbp-0x28],rdx
   0x26bf27 <+39>:    mov    r14,QWORD PTR [rbx+0x568]
   0x26bf2e <+46>:    test   r14,r14
   0x26bf31 <+49>:    jne    0x26bf42 <Module::SetDynamicIL(unsigned int, unsigned long, int)+66>
   0x26bf33 <+51>:    mov    rdi,rbx
   0x26bf36 <+54>:    call   0x26be90 <Module::InitializeDynamicILCrst()>
   0x26bf3b <+59>:    mov    r14,QWORD PTR [rbx+0x568]
   0x26bf42 <+66>:    mov    rdi,r14
   0x26bf45 <+69>:    call   0xcf150 <CrstBase::Enter()>
   0x26bf4a <+74>:    lea    rax,[rbx+0x578]
   0x26bf51 <+81>:    add    rbx,0x570
   0x26bf58 <+88>:    test   r15d,r15d
   0x26bf5b <+91>:    cmovne rbx,rax
   0x26bf5f <+95>:    mov    rdi,QWORD PTR [rbx]
   0x26bf62 <+98>:    test   rdi,rdi
   0x26bf65 <+101>:   jne    0x26bf85 <Module::SetDynamicIL(unsigned int, unsigned long, int)+133>
   0x26bf67 <+103>:   mov    edi,0x18
   0x26bf6c <+108>:   call   0xa4100 <operator new(unsigned long)>
   0x26bf71 <+113>:   mov    rdi,rax
   0x26bf74 <+116>:   xorps  xmm0,xmm0
   0x26bf77 <+119>:   movups XMMWORD PTR [rdi],xmm0
   0x26bf7a <+122>:   mov    QWORD PTR [rdi+0x10],0x0
   0x26bf82 <+130>:   mov    QWORD PTR [rbx],rdi
   0x26bf85 <+133>:   lea    rsi,[rbp-0x30]
   0x26bf89 <+137>:   call   0x2768b0 <SHash<DynamicILBlobTraits>::AddOrReplace(DynamicILBlobEntry const&)>
   0x26bf8e <+142>:   mov    rdi,r14
   0x26bf91 <+145>:   call   0xcf0c0 <CrstBase::Leave()>
   0x26bf96 <+150>:   mov    rax,QWORD PTR fs:0x28
   0x26bf9f <+159>:   cmp    rax,QWORD PTR [rbp-0x20]
   0x26bfa3 <+163>:   jne    0x26bfb0 <Module::SetDynamicIL(unsigned int, unsigned long, int)+176>
   0x26bfa5 <+165>:   add    rsp,0x18
   0x26bfa9 <+169>:   pop    rbx
   0x26bfaa <+170>:   pop    r14
   0x26bfac <+172>:   pop    r15
   0x26bfae <+174>:   pop    rbp
   0x26bfaf <+175>:   ret
   0x26bfb0 <+176>:   call   0xa0ac0 <__stack_chk_fail@plt>
   0x26bfb5 <+181>:   mov    rdi,rax
   0x26bfb8 <+184>:   call   0xa3eb0 <__clang_call_terminate>
   0x26bfbd <+189>:   mov    rbx,rax
   0x26bfc0 <+192>:   mov    rdi,r14
   0x26bfc3 <+195>:   call   0xcf0c0 <CrstBase::Leave()>
   0x26bfc8 <+200>:   mov    rdi,rbx
   0x26bfcb <+203>:   call   0xa0af0 <_Unwind_Resume@plt>
   0x26bfd0 <+208>:   mov    rdi,rax
   0x26bfd3 <+211>:   call   0xa3eb0 <__clang_call_terminate>
```
> *[disassembly.asm view raw](https://gist.githubusercontent.com/kevingosse/61be334d2da3464663bb9c8f82d1a29a/raw/0f421f74e387f483bb0620028ee2bc7aac3d94eb/disassembly.asm)*

OK, that’s **a lot** to process. Especially if, like me, you’re not that familiar with native disassembly. The trick is to compare it with the original source code (that’s why I posted `[SetDynamicIL](https://gist.githubusercontent.com/kevingosse/8c2d5b6301930d9dff9e613f7c980ec7)` earlier), and focus exclusively on what you’re looking for.

First, we need to locate the `this` parameter. Object-oriented programming does not exist at the assembly level, so the `this` pointer that we magically use must be given to the target function somehow. By convention, when calling an instance method, `this` is the first argument of the function.

Next, we need to know how arguments are given to the function. [Calling Wikipedia to the rescue](https://en.wikipedia.org/wiki/X86_calling_conventions), we learn that Linux uses the “System V AMD64 ABI” calling convention. In that convention, the first argument of a function is stored in the `rdi` register.

Now we need some kind of “anchor”. A well-identified point within the function that we can focus on. Right at the beginning of `SetDynamicIL`, we find this condition:

```
    if (m_debuggerSpecificData.m_pDynamicILCrst == NULL)
    {
        InitializeDynamicILCrst();
    }
```
> *[ceeload.cpp view raw](https://gist.githubusercontent.com/kevingosse/767650da40409cc70e34f63ba4623cdd/raw/e14e10ba06915c3d5ffad415157a1112f082eda5/ceeload.cpp)*

This is great because it uses `m_debuggerSpecificData` (the field we're looking for), it has a condition, and it calls a method (`InitializeDynamicILCrst`). This makes it very easy to spot in the disassembly. Now we know we have to focus on this bit:

```
   0x26bf00 <+0>:     push   rbp
   0x26bf01 <+1>:     mov    rbp,rsp
   0x26bf04 <+4>:     push   r15
   0x26bf06 <+6>:     push   r14
   0x26bf08 <+8>:     push   rbx
   0x26bf09 <+9>:     sub    rsp,0x18
   0x26bf0d <+13>:    mov    r15d,ecx
   0x26bf10 <+16>:    mov    rbx,rdi
   0x26bf13 <+19>:    mov    rax,QWORD PTR fs:0x28
   0x26bf1c <+28>:    mov    QWORD PTR [rbp-0x20],rax
   0x26bf20 <+32>:    mov    DWORD PTR [rbp-0x30],esi
   0x26bf23 <+35>:    mov    QWORD PTR [rbp-0x28],rdx
   0x26bf27 <+39>:    mov    r14,QWORD PTR [rbx+0x568]
   0x26bf2e <+46>:    test   r14,r14
   0x26bf31 <+49>:    jne    0x26bf42 <Module::SetDynamicIL(unsigned int, unsigned long, int)+66>
   0x26bf33 <+51>:    mov    rdi,rbx
   0x26bf36 <+54>:    call   0x26be90 <Module::InitializeDynamicILCrst()>
```
> *[disassembly.asm view raw](https://gist.githubusercontent.com/kevingosse/bcf7ee39ad318a92ded257396230d0bf/raw/1b6d860e2c644a535e9925bf219823516a8619d9/disassembly.asm)*

Remember, `this` is stored in the `rdi` register. This register is copied to `rbx` :

```
   0x26bf10 <+16>:    mov    rbx,rdi
```
> *[disassembly.asm view raw](https://gist.githubusercontent.com/kevingosse/3cfae7c0642104a95482cda85cca9f6f/raw/903f2c5b2c47cb60ac2d368fef4e61824471a212/disassembly.asm)*

Then we reuse this register here:

```
   0x26bf27 <+39>:    mov    r14,QWORD PTR [rbx+0x568]
   0x26bf2e <+46>:    test   r14,r14
   0x26bf31 <+49>:    jne    0x26bf42 <Module::SetDynamicIL(unsigned int, unsigned long, int)+66>
```
> *[disassembly.asm view raw](https://gist.githubusercontent.com/kevingosse/34014e81222e0d1a25aae8eeaf07d03c/raw/7fea193763fa3db293f27e043f9913cf14b54d8c/disassembly.asm)*

This code reads the memory at the address `rbx+0x568`, pushes the contents to the `r14` register, then tests something: `test r14,r14`. Testing a register against itself is the assembly way of checking if a value is empty. That’s our `if (m_debuggerSpecificData.m_pDynamicILCrst == NULL)` check! This means that `m_debuggerSpecificData.m_pDynamicILCrst` is located at the offset `0x568` from the address of the module instance.

That’s great, but I needed `m_debuggerSpecificData.m_pDynamicILBlobTable`, not `m_debuggerSpecificData.m_pDynamicILCrst`. So I had a look at the structure stored in the `m_debuggerSpecificData` field:

```
    struct DebuggerSpecificData
    {
        // Mutex protecting update access to the DynamicILBlobTable and TemporaryILBlobTable
        PTR_Crst                 m_pDynamicILCrst;

                                                // maps tokens for EnC/dynamics/reflection emit to their corresponding IL blobs
                                                // this map *always* overrides the Metadata RVA
        PTR_DynamicILBlobTable   m_pDynamicILBlobTable;

                                                // maps tokens for to their corresponding overriden IL blobs
                                                // this map conditionally overrides the Metadata RVA and the DynamicILBlobTable
        PTR_DynamicILBlobTable   m_pTemporaryILBlobTable;

        // hash table storing any profiler-provided instrumented IL offset mapping
        PTR_ILOffsetMappingTable m_pILOffsetMappingTable;

        // Strict count of # of methods in this module that are JMC-enabled.
        LONG    m_cTotalJMCFuncs;

        // The default JMC status for methods in this module.
        // Individual methods can be overridden.
        bool    m_fDefaultJMCStatus;
    };

```
> *[ceeload.h view raw](https://gist.githubusercontent.com/kevingosse/556abda11799d4285c8b00931d0a72a8/raw/16d2981fcd9ca2242c03aeb1603a841058d112eb/ceeload.h)*

Fields are stored in memory in the same order as they are declared in the code. So `m_pDynamicILBlobTable` is the pointer stored right after `m_pDynamicILCrst` in the memory.

To test this, I first needed the address of the module containing `LoadBackendTypes`. If you scroll all the way back to where I called `dumpmd`, you can find it in the output:

```
```
> dumpmd 00007FCF16509B58

Method Name:          Npgsql.PostgresDatabaseInfo+<LoadBackendTypes>d__16.MoveNext()
[...]
Module:               00007fcf13ec67d8
[...]
```
```
> *[dumpmd.md view raw](https://gist.githubusercontent.com/kevingosse/1088259022176b1e4a676e514ee72020/raw/1ea2d038d886d849e89d07070a8c160f3f415d0b/dumpmd.md)*

I was looking for the content of the memory at the offset `0x568` of the Module, so I added that to the Module address to get `0x7FCF13EC67D8 + 0x568 = 0x7FCF13EC6D40`

I then used LLDB to dump the memory at that address:

```
(lldb) memory read --count 4 --size 8 --format x 7FCF13EC6D40
```

```
0x7fcf13ec6d40: 0x00007fce8c90c820 0x00007fce8c938380
0x7fcf13ec6d50: 0x0000000000000000 0x0000000000000000
```

Assuming my reasoning was correct, `0x00007fce8c90c820` would be the address of `m_pDynamicILCrst`, and `0x00007fce8c938380` the address of `m_pDynamicILBlobTable`. There was no way to be completely sure, but I could check if the values in memory matched the layout of the table in the source code:

```
    PTR_element_t m_table;                // pointer to table
    count_t       m_tableSize;            // allocated size of table
    count_t       m_tableCount;           // number of elements in table
    count_t       m_tableOccupied;        // number, includes deleted slots
    count_t       m_tableMax;             // maximum occupied count before reallocating
```
> *[shash.h view raw](https://gist.githubusercontent.com/kevingosse/36538592a22d52dd3851b3b8239b2540/raw/c7043f5ba7d7612a02277d7c5663cf91bbc171d3/shash.h)*

One pointer to a table, followed by 4 integers indicating the size and the occupancy of the table.

I first dumped the pointer:

```
(lldb) memory read --count 2 --size 8 --format x 0x00007fce8c938380
```

```
0x7fce8c938380: 0x00007fce8c938a90 0x000000010000000b
```

`0x00007fce8c938a90` sure looks like a pointer to the heap. Then I checked the integers (using `size 4` instead of `size 8` ):

```
(lldb) memory read --count 8 --size 4 --format x 0x00007fce8c938380
```

```
0x7fce8c938380: 0x8c938a90 0x00007fce 0x0000000b 0x00000001
0x7fce8c938390: 0x00000001 0x00000008 0x00000045 0x00000000
```

The pointer was still there (looking backward because of [endianness](https://en.wikipedia.org/wiki/Endianness)), then a few small values. I mapped everything to the fields of the table and got:

```
m_table         = 0x00007fce8c938a90
m_tableSize     = 0x0b
m_tableCount    = 0x1
m_tableOccupied = 0x1
m_tableMax      = 0x8
```

Once again, there was no way to be sure, but the values were consistent with the expected layout, and seemed to indicate that there was one element stored in the table!

`m_table` is a hashtable associating method tokens and pointers to IL code:

```
struct  DynamicILBlobEntry
{
    mdToken     m_methodToken;
    TADDR       m_il;
};
```
> *[ceeload.h view raw](https://gist.githubusercontent.com/kevingosse/ea14526059184c260bb886c5159b4071/raw/ad7ff841f15d9dcf13f5059e92d9479b51435fe6/ceeload.h)*

I must confess I had a bit of trouble figuring out the layout of the hashtable from the source code (full of templates and other C++ magic), so I cheated a bit. I knew from the `dumpmd` output that my method token was `0x6000D24`. So I just dumped a bunch of memory at the memory location of the hashtable and looked for that value:

```
(lldb) memory read --count 32 --size 4 --format x 0x00007fce8c938a90
0x7fce8c938a90: 0x00000000 0x00007fce 0x00000000 0x00000000
0x7fce8c938aa0: 0x00000000 0x00007fcf 0x00000000 0x00000000
0x7fce8c938ab0: 0x00000000 0x6265645b 0x00000000 0x00000000
0x7fce8c938ac0: 0x00000000 0x6974616c 0x00000000 0x00000000
0x7fce8c938ad0: 0x00000000 0x74636e75 0x00000000 0x00000000
0x7fce8c938ae0: 0x00000000 0x36303437 0x00000000 0x00000000
0x7fce8c938af0: 0x06000d24 0x00000000 0x8c983790 0x00007fce
0x7fce8c938b00: 0x00000000 0x7367704e 0x00000000 0x00000000
```

It turned out that the value was next to a pointer (`0x00007fce8c983790`, backwards), so there was a good probability that it was pointing to the IL I was looking for!

How to confirm it? Every IL method has a header, so I decompiled the original `PostgresDatabaseInfo.LoadBackendTypes` method with [dnSpy](https://github.com/0xd4d/dnSpy) to look for a remarkable value. The `LocalVarSig` token had a value of `0x11000275`:

![][image_ref_MSprWVZVOS1CeVNBMmhGZFBfMVY1QU5BLnBuZw==]

I then dumped a few bytes at the address I found and looked for the value:

```
(lldb) memory read --count 8 --size 4 --format x 0x00007fce8c983790
```

```
0x7fce8c983790: 0x0002301b 0x000005fe 0x11000275 0x08007b02
0x7fce8c9837a0: 0x020a0400 0x0008037b 0x19060b04 0x0d167936
```

And sure enough, it matched!

The next and final step was to dump the IL and try to understand why it was causing the `InvalidProgramException`. That will be the subject of the next article.


[image_ref_MSprWVZVOS1CeVNBMmhGZFBfMVY1QU5BLnBuZw==]: data:image/png;base64,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
