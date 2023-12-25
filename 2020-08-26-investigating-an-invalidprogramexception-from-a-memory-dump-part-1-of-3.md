---
url: https://medium.com/@kevingosse/investigating-an-invalidprogramexception-from-a-memory-dump-part-1-of-3-bce634460cc3
canonical_url: https://medium.com/@kevingosse/investigating-an-invalidprogramexception-from-a-memory-dump-part-1-of-3-bce634460cc3
title: Investigating an InvalidProgramException from a memory dump (part 1 of 3)
subtitle: The first part of the investigation is an introduction to using a memory
  dump from a .NET application to find the information you seek.
slug: investigating-an-invalidprogramexception-from-a-memory-dump-part-1-of-3
description: ""
tags:
- csharp
- dotnet
- debugging
- software-development
- programming
author: Kevin Gosse
username: kevingosse
---

# Investigating an InvalidProgramException from a memory dump (part 1 of 3)

Datadog automated instrumentation for .NET works by rewriting the IL of interesting methods to emit traces that are then sent to the back-end. This is a complex piece of logic, written [using the profiler API](https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/profiling-overview?WT.mc_id=DT-MVP-5003493), and ridden with corner-cases. And as always with complex code, bugs are bound to happen, and those can be very difficult to diagnose.

As it turns out, we had customer reports of applications throwing `InvalidProgramException` when using our instrumentation. This exception is thrown when the JIT encounters invalid IL code, most likely emitted by our profiler. The symptoms were always the same: upon starting, the application had a random probability of ending up in a state where the exception would be thrown every time a method in particular was called. When that happened, restarting the application fixed the issue. The affected method would change from one time to the other. The issue was bad enough that the customers felt the need to report it, and rare enough that it couldn’t be reproduced at will. Yikes.

Since we couldn’t reproduce the issue ourselves, I decided to ask for a memory dump, and received one a few weeks later (random issues are random). This was my first time debugging this kind of problem, and it proved to be quite dodgy, so I figured out it would make a nice subject for an article.

The article ended up being **a lot** longer than I thought, so I divided it into 3 different parts.

* Part 1: Preliminary exploration

* [Part 2: Finding the generated IL](https://medium.com/@kevingosse/investigating-an-invalidprogramexception-from-a-memory-dump-part-2-of-3-daaecd8f3cf4)

* [Part 3: Identifying the error and fixing the bug](https://medium.com/@kevingosse/investigating-an-invalidprogramexception-from-a-memory-dump-part-3-of-3-c1d912075cb1)

# Preliminary exploration

The memory dump had been captured on a Linux instance. I opened it with [dotnet-dump](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-dump?WT.mc_id=DT-MVP-5003493), running on WSL2. The first step was to find what method was throwing the exception. Usually, this kind of memory dump is captured when the first-chance exception is thrown, and that exception is visible in the last column when using the `clrthreads` command. But I couldn’t find any:

```
```
> clrthreads

ThreadCount:      19
UnstartedThread:  0
BackgroundThread: 10
PendingThread:    0
DeadThread:       7
Hosted Runtime:   no
                                                                                                            Lock
 DBG   ID     OSID ThreadOBJ           State GC Mode     GC Alloc Context                  Domain           Count Apt Exception
   0    1        1 0000000001B318D0  2020020 Preemptive  0000000000000000:0000000000000000 0000000001AF60B0 0     Ukn
   5    2        a 0000000001AD1FE0    21220 Preemptive  00007FCE9864C630:00007FCE9864E2D0 0000000001AF60B0 0     Ukn (Finalizer)
   7    3        c 00007FCE88000C50  1020220 Preemptive  0000000000000000:0000000000000000 0000000001AF60B0 0     Ukn (Threadpool Worker)
   9    6       11 00007FCE8C17B250    21220 Preemptive  0000000000000000:0000000000000000 0000000001AF60B0 0     Ukn
  10   10       15 0000000002F8FAF0    21220 Preemptive  0000000000000000:0000000000000000 0000000001AF60B0 0     Ukn
  11   11       16 0000000002ECF840  2021020 Preemptive  00007FCE98650E48:00007FCE986522D0 0000000001AF60B0 0     Ukn
XXXX   29        0 00007FCE44002BE0  1031820 Preemptive  0000000000000000:0000000000000000 0000000001AF60B0 0     Ukn (Threadpool Worker)
XXXX   24        0 00007FCE5C006F60  1031820 Preemptive  0000000000000000:0000000000000000 0000000001AF60B0 0     Ukn (Threadpool Worker)
XXXX   16        0 00007FCE800017C0  1031820 Preemptive  0000000000000000:0000000000000000 0000000001AF60B0 0     Ukn (Threadpool Worker)
XXXX    4        0 00007FCE80002830  1031820 Preemptive  0000000000000000:0000000000000000 0000000001AF60B0 0     Ukn (Threadpool Worker)
XXXX   23        0 00007FCE5C020CF0  1031820 Preemptive  0000000000000000:0000000000000000 0000000001AF60B0 0     Ukn (Threadpool Worker)
XXXX   20        0 00007FCE7403F840  1031820 Preemptive  0000000000000000:0000000000000000 0000000001AF60B0 0     Ukn (Threadpool Worker)
XXXX   15        0 00007FCE5803B2F0  1031820 Preemptive  0000000000000000:0000000000000000 0000000001AF60B0 0     Ukn (Threadpool Worker)
  12   27      6d6 00007FCE74011A10  1021220 Preemptive  00007FCE98C7F330:00007FCE98C7FA28 0000000001AF60B0 0     Ukn (Threadpool Worker)
  13   21      700 00007FCE740A0030  1021220 Preemptive  00007FCE98D10578:00007FCE98D11168 0000000001AF60B0 0     Ukn (Threadpool Worker)
   8   28       10 00007FCE78000C50    20220 Preemptive  0000000000000000:0000000000000000 0000000001AF60B0 0     Ukn
  14   26      71b 00007FCE3C007610  1021220 Preemptive  00007FCE98E81908:00007FCE98E831C8 0000000001AF60B0 0     Ukn (Threadpool Worker)
  15   25      71d 00007FCE3C008750  1021220 Preemptive  00007FCE98E7BAC8:00007FCE98E7D1C8 0000000001AF60B0 0     Ukn (Threadpool Worker)
  16   17      71f 00007FCE400292B0  1021220 Preemptive  00007FCE98CFB800:00007FCE98CFD168 0000000001AF60B0 0     Ukn (Threadpool Worker)
  ```
```
> *[clrthreads.md view raw](https://gist.githubusercontent.com/kevingosse/d2b54559b340b66fa2f707e3c8fc30f5/raw/5d033a381df0fc2c52f595eb08ead02efd06ae2b/clrthreads.md)*

I then decided to have a look at the notes sent along with the dump (yeah, I know, I should have started there), and understood why I couldn’t see the exception: the customer confirmed that the issue was occurring and just captured the memory dump at a random point in time. Can’t blame them: I don’t even know how to capture a memory dump on first-chance exceptions on Linux, and [it doesn’t seem to be supported by procdump](https://github.com/microsoft/ProcDump-for-Linux). If somebody from the .NET diagnostics team reads me…

That’s OK though. If no garbage collection happened since the exception was thrown, it should still be hanging somewhere around on the heap. To find out, I used the `dumpheap -stat` command:

```
```
> dumpheap -stat -type InvalidProgramException

Statistics:
              MT    Count    TotalSize Class Name
00007fcf109e7428        3          384 System.InvalidProgramException
Total 3 objects
```
```
> *[dumpheap -stat.md view raw](https://gist.githubusercontent.com/kevingosse/ce58b1f714d77e09410410bd7ca527dd/raw/f81a0f9776e92875eb4f10c719b3378e04f1ac86/dumpheap%20-stat.md)*

Three of them, great. I used the `dumpheap -mt` command to get their address, with the value from the “MT” column. Then I used the `printexception` (`pe`) command to get the stacktrace associated to the exception:

```
```
> dumpheap -mt 00007fcf109e7428

         Address               MT     Size
00007fce985a4358 00007fcf109e7428      128
00007fce98a8b008 00007fcf109e7428      128
00007fce98b560f0 00007fcf109e7428      128

Statistics:
              MT    Count    TotalSize Class Name
00007fcf109e7428        3          384 System.InvalidProgramException
Total 3 objects


> pe 00007fce985a4358

Exception object: 00007fce985a4358
Exception type:   System.InvalidProgramException
Message:          Common Language Runtime detected an invalid program.
InnerException:   <none>
StackTrace (generated):
    SP               IP               Function
    00007FCE95127800 00007FCF1636CDC4 Customer.DataAccess.dll!Customer.DataAccess.Generic.AsyncDataAccessBase`2+<>c__DisplayClass1_0[[System.__Canon, System.Private.CoreLib],[System.__Canon, System.Private.CoreLib]].<CreateConnectionAsync>b__0(System.Threading.Tasks.Task)+0xd4
    00007FCE95127830 00007FCF1636CC4E System.Private.CoreLib.dll!System.Threading.Tasks.ContinuationResultTaskFromTask`1[[System.__Canon, System.Private.CoreLib]].InnerInvoke()+0x6e
    00007FCE95127870 00007FCF15BA54D1 System.Private.CoreLib.dll!System.Threading.ExecutionContext.RunFromThreadPoolDispatchLoop(System.Threading.Thread, System.Threading.ExecutionContext, System.Threading.ContextCallback, System.Object)+0x41
    00007FCE951278A0 00007FCF15CA33BC System.Private.CoreLib.dll!System.Runtime.ExceptionServices.ExceptionDispatchInfo.Throw()+0x1c
    00007FCE951278B0 00007FCF15BA51CE System.Private.CoreLib.dll!System.Threading.Tasks.Task.ExecuteWithThreadLocal(System.Threading.Tasks.Task ByRef, System.Threading.Thread)+0x17e
    00007FCE95127680 00007FCF15CA33BC System.Private.CoreLib.dll!System.Runtime.ExceptionServices.ExceptionDispatchInfo.Throw()+0x1c
    00007FCE95127690 00007FCF15BA4B3B System.Private.CoreLib.dll!System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(System.Threading.Tasks.Task)+0x4b
    00007FCE951276B0 00007FCF15CB4FF3 System.Private.CoreLib.dll!System.Runtime.CompilerServices.TaskAwaiter`1[[System.__Canon, System.Private.CoreLib]].GetResult()+0x13
    00007FCE951276D0 00007FCF16313EA5 Customer.DataAccess.dll!Customer.DataAccess.Generic.AsyncDataAccessBase`2+<CreateConnectionAsync>d__1[[System.__Canon, System.Private.CoreLib],[System.__Canon, System.Private.CoreLib]].MoveNext()+0x245
[... removed for brevity]
    00007FCE95112A00 00007FCF15BBB442 Microsoft.AspNetCore.Server.Kestrel.Core.dll!Microsoft.AspNetCore.Server.Kestrel.Core.Internal.Http.HttpProtocol+<ProcessRequests>d__214`1[[System.__Canon, System.Private.CoreLib]].MoveNext()+0x472

StackTraceString: <none>
HResult: 8013153a
```
```
> *[printexception.md view raw](https://gist.githubusercontent.com/kevingosse/e2cc287b99d7e1767e64f79b6c1b3a3f/raw/ce48ef0ba622f1a977e6cbb120528a491b6c2cee/printexception.md)*

> Note: Since this is the output from an actual memory dump sent by a customer, I redacted the namespaces containing business code and replaced by “Customer”

We see that the exception was thrown from `Customer.DataAccess.Generic.AsyncDataAccessBase`2+<>c__DisplayClass1_0.<CreateConnectionAsync>b__0`. The `<>c__` indicates a closure, so this was probably a lambda function declared in the `CreateConnectionAsync` method.

Since I didn’t have the source code, I used the`ip2md` command to convert the instruction pointer (second column of the stacktrace) to a MethodDescriptor, then fed it to the `dumpil` command to print the raw IL:

```
```
> ip2md 00007FCF1636CDC4

MethodDesc:   00007fcf161d3c78
Method Name:          Customer.DataAccess.Generic.AsyncDataAccessBase`2+<>c__DisplayClass1_0[[System.__Canon, System.Private.CoreLib],[System.__Canon, System.Private.CoreLib]].<CreateConnectionAsync>b__0(System.Threading.Tasks.Task)
Class:                00007fcf161c74d0
MethodTable:          00007fcf161d3ca0
mdToken:              000000000600009E
Module:               00007fcf1457e730
IsJitted:             yes
Current CodeAddr:     00007fcf1636ccf0
Version History:
  ILCodeVersion:      0000000000000000
  ReJIT ID:           0
  IL Addr:            0000000000000000
     CodeAddr:           00007fcf1636ccf0  (OptimizedTier1)
     NativeCodeVersion:  00007FCE5806D280
     CodeAddr:           00007fcf16276f20  (QuickJitted)
     NativeCodeVersion:  0000000000000000
     
     
> dumpil 00007fcf161d3c78

ilAddr is 00007FCF0C04C3AC pImport is 000000000312B860
ilAddr = 00007FCF0C04C3AC
IL_0000: ldarg.1
IL_0001: callvirt class [netstandard]System.AggregateException System.Threading.Tasks.Task::get_Exception()
IL_0006: brtrue.s IL_0036
IL_0008: ldarg.0
[...]
```
```
> *[dumpil.md view raw](https://gist.githubusercontent.com/kevingosse/34016c5fd6253f16cb99a201c4730aea/raw/67b503803f3a83423df0edc993e5469da6a7e8b1/dumpil.md)*

I’m not going to show the full IL for obvious privacy reasons, but one thing struck me: the code was not calling any method that we instrument so there was no reason the IL would have been rewritten (and it turns out there was no traces of rewriting).

Digging deeper, I got a hint at what was happening:

```
IL_0036: ldarg.1
IL_0037: callvirt class [netstandard]System.AggregateException System.Threading.Tasks.Task::get_Exception()
IL_003c: callvirt class [netstandard]System.Collections.ObjectModel.ReadOnlyCollection`1<class [netstandard]System.Exception> System.AggregateException::get_InnerExceptions()
IL_0041: callvirt int32 ::get_Count()
IL_0046: ldc.i4.1
IL_0047: bne.un.s IL_005b
IL_0049: ldarg.1
IL_004a: callvirt class [netstandard]System.AggregateException System.Threading.Tasks.Task::get_Exception()
IL_004f: callvirt class [netstandard]System.Collections.ObjectModel.ReadOnlyCollection`1<class [netstandard]System.Exception> System.AggregateException::get_InnerExceptions()
IL_0054: ldc.i4.0
IL_0055: callvirt !0 ::get_Item(int32)
IL_005a: throw
IL_005b: ldarg.1
IL_005c: callvirt class [netstandard]System.AggregateException System.Threading.Tasks.Task::get_Exception()
IL_0061: throw
```
> *[IL.il view raw](https://gist.githubusercontent.com/kevingosse/c9c99f4d2215f9262cb8486d541ec0f5/raw/18feecedb1806560b8ec1e743bb3067ea4712feb/IL.il)*

Translated into C#, this is pretty much equivalent to:

```
if (task.Exception.InnerExceptions.Count == 1)
{
	throw task.Exception.InnerExceptions[0];
}
else 
{
	throw task.Exception;
}
```
> *[exception.cs view raw](https://gist.githubusercontent.com/kevingosse/77c0e599a8e5a037a905b4c3ac6f8090/raw/82a23358445ed143285f6afffe58e50432c4015a/exception.cs)*

Basically, the method was there to log a few things (that I didn’t include in the snippet) then rethrow the exception. But since they didn’t wrap the exception before rethrowing it (and didn’t use `[ExceptionDispatchInfo](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.exceptionservices.exceptiondispatchinfo?view=netcore-3.1&WT.mc_id=DT-MVP-5003493)`), it overwrote the original callstack!

The caller code would have been something like:

```
await SomethingAsync().ContinueWith(task => 
{
	// Logging

	// Rethrow
	if (task.Exception.InnerExceptions.Count == 1)
	{
		throw task.Exception.InnerExceptions[0];
	}
	else 
	{
		throw task.Exception;
	}
});
```
> *[caller.cs view raw](https://gist.githubusercontent.com/kevingosse/5ffcce03b08314cb843f05e0fb4b19ae/raw/dba38c790ecd91f988ea762fdef526df9307df52/caller.cs)*

The usage of `ContinueWith` is confirmed by the presence of `ContinuationResultTaskFromTask`1` in the callstack.

That was really bad for me, because it meant that the original exception could have been thrown anywhere in whatever methods were called by `SomethingAsync`.

By looking at the raw IL of the caller method, I figured out that `SomethingAsync` was `System.Data.Common.DbConnection::OpenAsync`. Since the application used PostgreSQL with the [Npgsql library](https://www.npgsql.org/), and since our tracer automatically instruments that library, it made sense that the rewritten method would be somewhere in there. Normally, I could have checked our logs to quickly find what Npgsql method was rewritten, but the customer didn’t retrieve them before killing the instance, and waiting for the problem to happen again could have taken weeks (random issue is random). So I decided to bite the bullet and start the painstaking process of cross-checking the source code of Npgsql with the state of the objects in the memory dump to find the exact place where the execution stopped and the exception was originally thrown.

For instance, at some point the method `PostgresDatabaseInfoFactory.Load` is called:

```
        public async Task<NpgsqlDatabaseInfo?> Load(NpgsqlConnection conn, NpgsqlTimeout timeout, bool async)
        {
            var db = new PostgresDatabaseInfo(conn);
            await db.LoadPostgresInfo(conn, timeout, async);
            Debug.Assert(db.LongVersion != null);
            return db;
        }
```
> *[PostgresDatabaseInfo.cs view raw](https://gist.githubusercontent.com/kevingosse/c587cb102a1748904aa0db170f517a4e/raw/fe24e47998031f43d403963bcaf735bc12969269/PostgresDatabaseInfo.cs)*

There were instances of `PostgresDatabaseInfo` on the heap, so I knew this method ran properly. From there, the `LoadBackendTypes` method is called, and the result is assigned to the `_types` field:

```
        internal async Task LoadPostgresInfo(NpgsqlConnection conn, NpgsqlTimeout timeout, bool async)
        {
            HasIntegerDateTimes =
                conn.PostgresParameters.TryGetValue("integer_datetimes", out var intDateTimes) &&
                intDateTimes == "on";

            IsRedshift = conn.Settings.ServerCompatibilityMode == ServerCompatibilityMode.Redshift;
            _types = await LoadBackendTypes(conn, timeout, async);
        }
```
> *[PostgresDatabaseInfo.cs view raw](https://gist.githubusercontent.com/kevingosse/56779ea473ac2f477604a38b8c30051b/raw/f5ede9835d4053c91561d0058463bb6cf12489e0/PostgresDatabaseInfo.cs)*

But when inspecting the instances of `PostgresDatabaseInfo` (using the `dumpobj` or `do` command with the address returned by the `dumpheap -mt` command), we can see that the `_types` field has no value:

```
```
> dumpheap -stat -type PostgresDatabaseInfo

Statistics:
              MT    Count    TotalSize Class Name
00007fcf162568f0        1           24 Npgsql.PostgresDatabaseInfoFactory
00007fcf16256b10        2          256 Npgsql.PostgresDatabaseInfo
Total 3 objects


> dumpheap -mt 00007fcf16256b10

         Address               MT     Size
00007fce98a8adb0 00007fcf16256b10      128
00007fce98b55e98 00007fcf16256b10      128

Statistics:
              MT    Count    TotalSize Class Name
00007fcf16256b10        2          256 Npgsql.PostgresDatabaseInfo
Total 2 objects


> do 00007fce98a8adb0

Name:        Npgsql.PostgresDatabaseInfo
MethodTable: 00007fcf16256b10
EEClass:     00007fcf16245848
Size:        128(0x80) bytes
File:        /Npgsql.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007fcf0f640f90  40001e5        8        System.String  0 instance 00007fce984d56f0 <Host>k__BackingField
00007fcf0f63a0e8  40001e6       68         System.Int32  1 instance             5432 <Port>k__BackingField
00007fcf0f640f90  40001e7       10        System.String  0 instance 00007fce984d5750 <Name>k__BackingField
00007fcf0f75e410  40001e8       18       System.Version  0 instance 00007fce98a8ae30 <Version>k__BackingField
00007fcf0f6360c8  40001e9       6c       System.Boolean  1 instance                1 <HasIntegerDateTimes>k__BackingField
00007fcf0f6360c8  40001ea       6d       System.Boolean  1 instance                1 <SupportsTransactions>k__BackingField
00007fcf16546df8  40001eb       20 ...aseType, Npgsql]]  0 instance 00007fce98a8ae70 _baseTypesMutable
00007fcf16547328  40001ec       28 ...rayType, Npgsql]]  0 instance 00007fce98a8ae90 _arrayTypesMutable
00007fcf16547858  40001ed       30 ...ngeType, Npgsql]]  0 instance 00007fce98a8aeb0 _rangeTypesMutable
00007fcf16547d88  40001ee       38 ...numType, Npgsql]]  0 instance 00007fce98a8aed0 _enumTypesMutable
00007fcf165482b8  40001ef       40 ...iteType, Npgsql]]  0 instance 00007fce98a8aef0 _compositeTypesMutable
00007fcf165487e8  40001f0       48 ...ainType, Npgsql]]  0 instance 00007fce98a8af10 _domainTypesMutable
00007fcf16548d18  40001f1       50 ...resType, Npgsql]]  0 instance 00007fce98a8af30 <ByOID>k__BackingField
00007fcf16549378  40001f2       58 ...resType, Npgsql]]  0 instance 00007fce98a8af78 <ByFullName>k__BackingField
00007fcf16549378  40001f3       60 ...resType, Npgsql]]  0 instance 00007fce98a8afc0 <ByName>k__BackingField
00007fcf162191f8  40001e3      1b0 ...aseInfo, Npgsql]]  0   static 00007fce984ecc48 Cache
00007fcf16545ed0  40001e4      1b8 ...Factory, Npgsql]]  0   static 00007fce984ece00 Factories
00007fcf16509488  400028e       70 ...resType, Npgsql]]  0 instance 0000000000000000 _types
00007fcf0f6360c8  400028f       6e       System.Boolean  1 instance                0 <IsRedshift>k__BackingField
00007fcf161d71c0  400028d      220 ...ging.NpgsqlLogger  0   static 0000000000000000 Log
```
```
> *[PostgresDatabaseInfo.md view raw](https://gist.githubusercontent.com/kevingosse/9a7fb1c4bdf22756a6cce571f7466e37/raw/fa9b916f4e4567f6a1c6ff53a68ea1705bc2f48d/PostgresDatabaseInfo.md)*

Therefore, the execution stopped somewhere in the `LoadBackendTypes` method. That method has calls to `NpgsqlCommand.ExecuteReader` and `NpgsqlCommand.ExecuteReaderAsync`, which are two method that our tracer instruments, and therefore is a candidate for rewriting.

Great! At this point I just had to dump the generated IL, find the error, and call it a day. Right? Well it got more complicated than I anticipated, as we’ll see in the next article.


