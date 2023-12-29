---
url: investigating-a-native-deadlock-in-a-net-linux-application-97979a005ebd
canonical_url: https://medium.com/@kevingosse/investigating-a-native-deadlock-in-a-net-linux-application-97979a005ebd
title: Debugging a native deadlock in a .NET Linux application
subtitle: Debugging a native deadlock in a .NET Linux application
date: 2021-02-09
description: ""
tags:
- dotnet
- debugging
- programming
- lldb
- software-development
author: Kevin Gosse
---

This story begins when one of our integrations tests started got stuck on one PR that seemingly impacted unrelated code. This is a nice excuse to cover some concepts I haven't touched in my previous articles, such as downloading the .NET symbols on Linux.

# Preliminary inspection

The failure was occurring in a Linux test. After a while, I managed to reproduce the issue locally in a docker container. Usually the first step would be to attach a debugger, but I didn't want to spend the time to find the commands to inject the debugger in the container. So I started by capturing a memory dump with dotnet-dump:

```bash
$ dotnet-dump collect -p <pid> --type Full
```

Note that a heap-only memory dump (`--type Heap`) should be enough in theory, but I've had so much trouble with those in the past that I now always capture full dump unless I have a compelling reason not to.

I then used the `dotnet-dump analyze` command to open it:

```bash
$ dotnet-dump analyze core_20210205_190420
Loading core dump: core_20210205_190420 ...
Ready to process analysis commands. Type 'help' to list available commands or 'help [command]' to get detailed help on a command.
Type 'quit' or 'exit' to exit the session.
>
```

The integration test was getting stuck, so I was looking for a deadlock of sorts. I started by using the `pstacks` command to get an overview of the callstacks:

```
> pstacks

________________________________________________
 ~~~~ c52
    1 System.Runtime.CompilerServices.AsyncMethodBuilderCore.Start(__Canon ByRef)
    1 System.Runtime.CompilerServices.AsyncTaskMethodBuilder.Start(__Canon ByRef)
    1 Samples.HttpMessageHandler.Program.Main(String[])
    1 Samples.HttpMessageHandler.Program.<Main>(String[])


________________________________________________
 ~~~~ c79
    1 Datadog.Trace.Agent.Api+<SendTracesAsync>d__9.MoveNext()
    1 System.Runtime.CompilerServices.AsyncMethodBuilderCore.Start(__Canon ByRef)
    1 System.Runtime.CompilerServices.AsyncTaskMethodBuilder<System.Boolean>.AsyncTaskMethodBuilder`1(__Canon ByRef)
    1 Datadog.Trace.Agent.Api.SendTracesAsync(Span[][])
    1 Datadog.Trace.Agent.AgentWriter.Ping()
    1 Datadog.Trace.Tracer+<WriteDiagnosticLog>d__54.MoveNext()
    1 System.Runtime.CompilerServices.AsyncMethodBuilderCore.Start(__Canon ByRef)
    1 System.Runtime.CompilerServices.AsyncTaskMethodBuilder.Start(__Canon ByRef)
    1 Datadog.Trace.Tracer.WriteDiagnosticLog()
    1 System.Threading.Tasks.Task<System.__Canon>.InnerInvoke()
    1 System.Threading.Tasks.Task+<>c.cctor>b__274_0(Object)
    1 System.Threading.ExecutionContext.RunFromThreadPoolDispatchLoop(Thread, ExecutionContext, ContextCallback, Object)
    1 System.Threading.Tasks.Task.ExecuteWithThreadLocal(Task ByRef, Thread)
    1 System.Threading.Tasks.Task.ExecuteEntryUnsafe(Thread)
    1 System.Threading.Tasks.Task.ExecuteFromThreadPool(Thread)
    1 System.Threading.ThreadPoolWorkQueue.Dispatch()
    1 System.Threading._ThreadPoolWaitCallback.PerformWaitCallback()


==> 2 threads with 2 roots
```

This revealed one thread executing `Datadog.Trace.Agent.Api.SendTracesAsync`. This method has no synchronization, so no obvious reason to get stuck. Just to be extra sure it was actually stuck (verifying your assumptions can save you a lot of time in the long run), I captured a second memory dump and confirmed that the same thread was stuck in the same method.

Looking for more information, I decided to have a closer look at the callstack using `clrstack`. `pstacks` indicated that the OS thread id was `c79`, unfortunately that's not a value I could use directly. Instead, I ran `clrthreads` to dump the list of managed threads:

```
> clrthreads

ThreadCount:      7
UnstartedThread:  0
BackgroundThread: 6
PendingThread:    0
DeadThread:       0
Hosted Runtime:   no
                                                                                                            Lock
 DBG   ID     OSID ThreadOBJ           State GC Mode     GC Alloc Context                  Domain           Count Apt Exception
   0    1      c52 0000559EA26B4180    20020 Preemptive  00007F2800061320:00007F2800061FD0 0000559EA25C8590 0     Ukn (GC)
   8    2      c5b 0000559EA26A5260    21220 Preemptive  0000000000000000:0000000000000000 0000559EA25C8590 0     Ukn (Finalizer)
  10    3      c5d 00007F27F4000C50  1020220 Preemptive  0000000000000000:0000000000000000 0000559EA25C8590 0     Ukn (Threadpool Worker)
  11    4      c5e 00007F27F4001C10  1021220 Preemptive  00007F280005CA50:00007F280005DFD0 0000559EA25C8590 0     Ukn (Threadpool Worker)
  13    5      c77 00007F27E8002DA0  1021220 Preemptive  00007F280005E2E8:00007F280005FFD0 0000559EA25C8590 0     Ukn (Threadpool Worker)
  15    6      c79 0000559EA2E8B990  1021222 Cooperative 00007F2800062D28:00007F2800063FD0 0000559EA25C8590 0     Ukn (Threadpool Worker)
  16    7      c7a 00007F27D80014F0  1021220 Preemptive  00007F2800064258:00007F2800065FD0 0000559EA25C8590 0     Ukn (Threadpool Worker)
```

I looked for the one with the value `c79` in the `OSID` column, then took the value of the `DBG` column (15). Then I could feed that value to the `setthread` command. An alternative is to convert the OS thread id to decimal (c79 = 3193), then use the `-t` parameter of the `setthread` command:

```
> setthread 15
> setthread -t 3193
```

Then I could run the `clrstack` command to get the detailed callstack:

```
> clrstack

OS Thread Id: 0xc79 (15)
        Child SP               IP Call Site
00007F288A3F4D10 00007f28a117129c [PrestubMethodFrame: 00007f288a3f4d10] Datadog.Trace.Agent.Transports.ApiWebRequestFactory.Create(System.Uri)
00007F288A3F4E80 00007F2827A3DDAD Datadog.Trace.Agent.Api+<SendTracesAsync>d__9.MoveNext()
00007F288A3F5100 00007F2827A3263F System.Runtime.CompilerServices.AsyncMethodBuilderCore.Start[[System.__Canon, System.Private.CoreLib]](System.__Canon ByRef) [/_/src/System.Private.CoreLib/shared/System/Runtime/CompilerServices/AsyncMethodBuilder.cs @ 1015]
00007F288A3F5180 00007F2827A3D595 System.Runtime.CompilerServices.AsyncTaskMethodBuilder`1[[System.Boolean, System.Private.CoreLib]].Start[[System.__Canon, System.Private.CoreLib]](System.__Canon ByRef) [/_/src/System.Private.CoreLib/shared/System/Runtime/CompilerServices/AsyncMethodBuilder.cs @ 350]
00007F288A3F51C0 00007F2827A3D46F Datadog.Trace.Agent.Api.SendTracesAsync(Datadog.Trace.Span[][])
00007F288A3F5210 00007F2827A3D364 Datadog.Trace.Agent.AgentWriter.Ping()
00007F288A3F5250 00007F2827A3BD8D Datadog.Trace.Tracer+<WriteDiagnosticLog>d__54.MoveNext()
00007F288A3F55D0 00007F2827A3263F System.Runtime.CompilerServices.AsyncMethodBuilderCore.Start[[System.__Canon, System.Private.CoreLib]](System.__Canon ByRef) [/_/src/System.Private.CoreLib/shared/System/Runtime/CompilerServices/AsyncMethodBuilder.cs @ 1015]
00007F288A3F5650 00007F2827A32595 System.Runtime.CompilerServices.AsyncTaskMethodBuilder.Start[[System.__Canon, System.Private.CoreLib]](System.__Canon ByRef) [/_/src/System.Private.CoreLib/shared/System/Runtime/CompilerServices/AsyncMethodBuilder.cs @ 223]
00007F288A3F5690 00007F2827A3908A Datadog.Trace.Tracer.WriteDiagnosticLog()
00007F288A3F56E0 00007F2827A31C2E System.Threading.Tasks.Task`1[[System.__Canon, System.Private.CoreLib]].InnerInvoke() [/_/src/System.Private.CoreLib/shared/System/Threading/Tasks/Future.cs @ 518]
00007F288A3F5760 00007F2827A316B1 System.Threading.Tasks.Task+<>c.<.cctor>b__274_0(System.Object) [/_/src/System.Private.CoreLib/shared/System/Threading/Tasks/Task.cs @ 2427]
00007F288A3F5790 00007F2827A313EF System.Threading.ExecutionContext.RunFromThreadPoolDispatchLoop(System.Threading.Thread, System.Threading.ExecutionContext, System.Threading.ContextCallback, System.Object) [/_/src/System.Private.CoreLib/shared/System/Threading/ExecutionContext.cs @ 289]
00007F288A3F5810 00007F2827A30D25 System.Threading.Tasks.Task.ExecuteWithThreadLocal(System.Threading.Tasks.Task ByRef, System.Threading.Thread) [/_/src/System.Private.CoreLib/shared/System/Threading/Tasks/Task.cs @ 2389]
00007F288A3F58D0 00007F2827A307B0 System.Threading.Tasks.Task.ExecuteEntryUnsafe(System.Threading.Thread) [/_/src/System.Private.CoreLib/shared/System/Threading/Tasks/Task.cs @ 2321]
00007F288A3F5900 00007F2827A3070F System.Threading.Tasks.Task.ExecuteFromThreadPool(System.Threading.Thread) [/_/src/System.Private.CoreLib/shared/System/Threading/Tasks/Task.cs @ 2312]
00007F288A3F5920 00007F2827A2EE7D System.Threading.ThreadPoolWorkQueue.Dispatch() [/_/src/System.Private.CoreLib/shared/System/Threading/ThreadPool.cs @ 663]
00007F288A3F5980 00007F2827A2E459 System.Threading._ThreadPoolWaitCallback.PerformWaitCallback() [/_/src/System.Private.CoreLib/src/System/Threading/ThreadPool.CoreCLR.cs @ 29]
00007F288A3F5D30 00007f28a02a6d0f [DebuggerU2MCatchHandlerFrame: 00007f288a3f5d30]
```

This uncovered an interesting piece of data: the thread was not actually stuck on `Datadog.Trace.Agent.Api.SendTracesAsync` but on a "`PrestubMethodFrame`" pointing to `Datadog.Trace.Agent.Transports.ApiWebRequestFactory.Create`. I guessed this was the stub used to trigger the JIT-compilation of the method. To confirm that hypothesis, I checked if the method was already JIT-compiled:

```
> name2ee Datadog.Trace.dll Datadog.Trace.Agent.Transports.ApiWebRequestFactory.Create

Module:      00007f2827457ab0
Assembly:    Datadog.Trace.dll
Token:       0000000006001C88
MethodDesc:  00007f28279df8e8
Name:        Datadog.Trace.Agent.Transports.ApiWebRequestFactory.Create(System.Uri)
Not JITTED yet. Use 'bpmd -md 00007F28279DF8E8' to break on run.
```

An alternative would have been to use the `dumpmt -md` command with the pointer to the method table:

```
> dumpmt -md 00007f28279df908

EEClass:         00007F2827AC7AD0
Module:          00007F2827457AB0
Name:            Datadog.Trace.Agent.Transports.ApiWebRequestFactory
mdToken:         00000000020003D6
File:            /project/test/test-applications/integrations/Samples.HttpMessageHandler/bin/Debug/netcoreapp3.0/publish/Datadog.Trace.dll
BaseSize:        0x18
ComponentSize:   0x0
DynamicStatics:  false
ContainsPointers false
Slots in VTable: 7
Number of IFaces in IFaceMap: 1
--------------------------------------
MethodDesc Table
           Entry       MethodDesc    JIT Name
00007F2826E90090 00007F282649C570   NONE System.Object.Finalize()
00007F2826E90098 00007F282649C580   NONE System.Object.ToString()
00007F2826E900A0 00007F282649C590   NONE System.Object.Equals(System.Object)
00007F2826E900B8 00007F282649C5D0   NONE System.Object.GetHashCode()
00007F28279F99E8 00007F28279DF8D8   NONE Datadog.Trace.Agent.Transports.ApiWebRequestFactory.Info(System.Uri)
00007F28279F99F0 00007F28279DF8E8   NONE Datadog.Trace.Agent.Transports.ApiWebRequestFactory.Create(System.Uri)
00007F2827A2A370 00007F28279DF8F8    JIT Datadog.Trace.Agent.Transports.ApiWebRequestFactory..ctor()
```

You can fetch the MT using `dumpmodule -mt <module address>`, and the module address using `dumpdomain`… Yeah, `name2ee` is usually faster.

Anyway, both `name2ee` and `dumpmt` revealed that the method wasn't JIT-compiled yet. Given that our product patches the code during JIT compilation, it wasn't really surprising to me that something could go wrong in that part of the process. In any case, it was time to switch to native debugging to get more information.

# One layer deeper

LLDB is the recommended debugger to use with the CLR on Linux, though GDB works fine in most scenarios.

On most Linux distributions, you can get LLDB using your package manager:

```bash
$ sudo apt install lldb
```

If not (for instance, if you're using CentOS), good luck [building it from the source](https://lldb.llvm.org/resources/build.html).

Even though I don't use it for this investigation, I also recommend [installing dotnet-sos](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-sos?WT.mc_id=DT-MVP-5003493) then running the `dotnet-sos install` command. This will update your LLDB profile to automatically load SOS, which is very convenient.

Then you can load the memory dump in LLDB:

```bash
$ lldb -c core_20210205_190420
Added Microsoft public symbol server
(lldb) target create --core "core_20210205_190420"
Core file '/temp/core_20210205_190420' (x86_64) was loaded.
```

I used the `thread backtrace` command to display the callstack of the first thread:

```
(lldb) thread backtrace

* thread #1, name = 'dotnet', stop reason = signal SIGABRT
  * frame #0: 0x00007f28a116e3f9 libpthread.so.0`__pthread_cond_timedwait + 761
    frame #1: 0x00007f28a04d070b libcoreclr.so`___lldb_unnamed_symbol14084$$libcoreclr.so + 299
    frame #2: 0x00007f28a04d0324 libcoreclr.so`___lldb_unnamed_symbol14083$$libcoreclr.so + 388
    frame #3: 0x00007f28a04d493f libcoreclr.so`___lldb_unnamed_symbol14151$$libcoreclr.so + 1743
    frame #4: 0x00007f28a04d4be9 libcoreclr.so`___lldb_unnamed_symbol14153$$libcoreclr.so + 89
    frame #5: 0x00007f28a0271064 libcoreclr.so`___lldb_unnamed_symbol5709$$libcoreclr.so + 228
    frame #6: 0x00007f28a0275263 libcoreclr.so`___lldb_unnamed_symbol5770$$libcoreclr.so + 931
    frame #7: 0x00007f28a02768fe libcoreclr.so`___lldb_unnamed_symbol5788$$libcoreclr.so + 446
    frame #8: 0x00007f28a018838c libcoreclr.so`___lldb_unnamed_symbol2479$$libcoreclr.so + 892
    frame #9: 0x00007f28a023f23c libcoreclr.so`___lldb_unnamed_symbol5125$$libcoreclr.so + 300
    frame #10: 0x00007f289c9ee613 Datadog.Trace.ClrProfiler.Native.so`trace::CorProfiler::CallTarget_RequestRejitForModule(unsigned long, trace::ModuleMetadata*, std::vector<trace::IntegrationMethod, std::allocator<trace::IntegrationMethod> > const&) + 4563
    frame #11: 0x00007f289c9ed248 Datadog.Trace.ClrProfiler.Native.so`trace::CorProfiler::ModuleLoadFinished(unsigned long, int) + 5544
    frame #12: 0x00007f28a01f7a79 libcoreclr.so`___lldb_unnamed_symbol4010$$libcoreclr.so + 105
[...]
```

There's a lot of noise but I could still deduce that code from `Datadog.Trace.ClrProfiler.Native` was waiting for… something in `libcoreclr.so`.

I then looked for the thread that we spotted in `dotnet-dump` (the one with the thread id (c79 / 3193) and inspected its callstack:

```
(lldb) thread list

Process 3154 stopped
* thread #1: tid = 3154, 0x00007f28a116e3f9 libpthread.so.0`__pthread_cond_timedwait + 761, name = 'dotnet', stop reason = signal SIGABRT
  thread #2: tid = 3156, 0x00007f28a0d6cf59 libc.so.6`syscall + 25, stop reason = signal SIGABRT
  thread #3: tid = 3157, 0x00007f28a0d6cf59 libc.so.6`syscall + 25, stop reason = signal SIGABRT
  thread #4: tid = 3158, 0x00007f28a0d67819 libc.so.6`__poll + 73, stop reason = signal SIGABRT
  thread #5: tid = 3159, 0x00007f28a1171d0e libpthread.so.0`__libc_open64 + 206, stop reason = signal SIGABRT
  thread #6: tid = 3160, 0x00007f28a116e00c libpthread.so.0`__pthread_cond_wait + 508, stop reason = signal SIGABRT
  thread #7: tid = 3161, 0x00007f28a116e00c libpthread.so.0`__pthread_cond_wait + 508, stop reason = signal SIGABRT
  thread #8: tid = 3162, 0x00007f28a116e35b libpthread.so.0`__pthread_cond_timedwait + 603, stop reason = signal SIGABRT
  thread #9: tid = 3163, 0x00007f28a116e3f9 libpthread.so.0`__pthread_cond_timedwait + 761, stop reason = signal SIGABRT
  thread #10: tid = 3164, 0x00007f28a11720ca libpthread.so.0`__waitpid + 74, stop reason = signal SIGABRT
  thread #11: tid = 3165, 0x00007f28a116e00c libpthread.so.0`__pthread_cond_wait + 508, stop reason = signal SIGABRT
  thread #12: tid = 3166, 0x00007f28a116e00c libpthread.so.0`__pthread_cond_wait + 508, stop reason = signal SIGABRT
  thread #13: tid = 3167, 0x00007f28a116e3f9 libpthread.so.0`__pthread_cond_timedwait + 761, stop reason = signal SIGABRT
  thread #14: tid = 3191, 0x00007f28a116e00c libpthread.so.0`__pthread_cond_wait + 508, stop reason = signal SIGABRT
  thread #15: tid = 3192, 0x00007f28a116e00c libpthread.so.0`__pthread_cond_wait + 508, stop reason = signal SIGABRT
  thread #16: tid = 3193, 0x00007f28a117129c libpthread.so.0`__lll_lock_wait + 28, stop reason = signal SIGABRT
  thread #17: tid = 3194, 0x00007f28a116e00c libpthread.so.0`__pthread_cond_wait + 508, stop reason = signal SIGABRT

(lldb) thread select 16

* thread #16, stop reason = signal SIGABRT
    frame #0: 0x00007f28a117129c libpthread.so.0`__lll_lock_wait + 28
libpthread.so.0`__lll_lock_wait:
->  0x7f28a117129c <+28>: movl   %edx, %eax
    0x7f28a117129e <+30>: xchgl  %eax, (%rdi)
    0x7f28a11712a0 <+32>: testl  %eax, %eax
    0x7f28a11712a2 <+34>: jne    0x7f28a1171295            ; <+21>

(lldb) thread backtrace

* thread #16, stop reason = signal SIGABRT
  * frame #0: 0x00007f28a117129c libpthread.so.0`__lll_lock_wait + 28
    frame #1: 0x00007f28a116a714 libpthread.so.0`__GI___pthread_mutex_lock + 84
    frame #2: 0x00007f289c9faf83 Datadog.Trace.ClrProfiler.Native.so`__gthread_mutex_lock(pthread_mutex_t*) + 35
    frame #3: 0x00007f289ca462b8 Datadog.Trace.ClrProfiler.Native.so`std::mutex::lock() + 24
    frame #4: 0x00007f289c9fc943 Datadog.Trace.ClrProfiler.Native.so`std::lock_guard<std::mutex>::lock_guard(std::mutex&) + 35
    frame #5: 0x00007f289c9eb449 Datadog.Trace.ClrProfiler.Native.so`trace::CorProfiler::AssemblyLoadFinished(unsigned long, int) + 185
    frame #6: 0x00007f28a01f82c9 libcoreclr.so`___lldb_unnamed_symbol4024$$libcoreclr.so + 105
    frame #7: 0x00007f28a02c1017 libcoreclr.so`___lldb_unnamed_symbol6779$$libcoreclr.so + 647
[...]
```

This thread was also executing code in `Datadog.Trace.ClrProfiler.Native`, and waited on a global lock that we use to protect our data structures. So it looked like the first thread was holding the lock and calling some code in `libcoreclr`, and the other thread was waiting for this lock to be freed. The last missing piece was figuring out why the code in `libcoreclr` was taking so long.

.NET Core ships without the internal symbols, but they can be downloaded [using the `dotnet-symbol` tool](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-symbol?WT.mc_id=DT-MVP-5003493). It's very simple to use, just indicate the location of the memory dump and the folder to write the symbols to:

```bash
$ dotnet-symbol /temp/core_20210205_190420 -o /symbols/
```

Downloading all the symbols will take a few minutes. You may get an error when downloading the symbols for `libcoreclr.so` ("Unable to read data from the transport connection: Operation canceled"). That's because the tool has a hardcoded 4 minutes timeout, and the symbol server is very slow when accessed outside of the United States. There is already [an issue to track that](https://github.com/dotnet/symstore/issues/246), until it's fixed I suggest you clone [the `dotnet/symstore` github repository](https://github.com/dotnet/symstore/), and [change the value of the timeout in the `HttpSymbolStore` class](https://github.com/dotnet/symstore/blob/98717c63ec8342acf8a07aa5c909b88bd0c664cc/src/Microsoft.SymbolStore/SymbolStores/HttpSymbolStore.cs#L68):

```csharp
            // Normal unauthenticated client
            _client = new HttpClient {
                // Timeout = TimeSpan.FromMinutes(4)
                Timeout = TimeSpan.FromMinutes(10)
            };
```

Then you can use the provided `build.sh` script to compile your changes, and you'll find your fixed version of dotnet-symbol in the `artifacts/bin/dotnet-symbol/` folder.

Then you can feed those symbols to LLDB:

```bash
$ lldb -O "settings set target.debug-file-search-paths /symbols/" -c core_20210205_190420
```

Or if like me you're too lazy to remember/find/type this `settings set target.debug-file-search-paths` thing, just copy all the symbols in the same folder as the memory dump and it'll work just fine.

With the symbols, I could get more information about what the first thread was doing:

```
(lldb) thread backtrace

* thread #1, name = 'dotnet', stop reason = signal SIGABRT
  * frame #0: 0x00007f28a116e3f9 libpthread.so.0`__pthread_cond_timedwait + 761
    frame #1: 0x00007f28a04d070b libcoreclr.so`CorUnix::CPalSynchronizationManager::ThreadNativeWait(ptnwdNativeWaitData=<unavailable>, dwTimeout=<unavailable>, ptwrWakeupReason=<unavailable>, pdwSignaledObject=<unavailable>) at synchmanager.cpp:484
    frame #2: 0x00007f28a04d0324 libcoreclr.so`CorUnix::CPalSynchronizationManager::BlockThread(this=0x0000559ea25b5280, pthrCurrent=0x0000559ea2586f70, dwTimeout=10, fAlertable=false, fIsSleep=<unavailable>, ptwrWakeupReason=0x00007ffc65d75338, pdwSignaledObject=<unavailable>) at synchmanager.cpp:302
    frame #3: 0x00007f28a04d493f libcoreclr.so`CorUnix::InternalWaitForMultipleObjectsEx(pThread=0x0000559ea2586f70, nCount=<unavailable>, lpHandles=<unavailable>, bWaitAll=<unavailable>, dwMilliseconds=<unavailable>, bAlertable=<unavailable>, bPrioritize=<unavailable>) at wait.cpp:640
    frame #4: 0x00007f28a04d4be9 libcoreclr.so`::WaitForSingleObjectEx(hHandle=<unavailable>, dwMilliseconds=10, bAlertable=NO) at wait.cpp:139
    frame #5: 0x00007f28a0271064 libcoreclr.so`CLREventBase::WaitEx(unsigned int, WaitMode, PendingSync*) [inlined] CLREventWaitHelper2(handle=<unavailable>, dwMilliseconds=<unavailable>, alertable=<unavailable>) at synch.cpp:377
    frame #6: 0x00007f28a027105f libcoreclr.so`CLREventBase::WaitEx(unsigned int, WaitMode, PendingSync*) [inlined] CLREventWaitHelper(void*, unsigned int, int)::$_1::operator()(CLREventWaitHelper(void*, unsigned int, int)::Param*) const at synch.cpp:402
    frame #7: 0x00007f28a0271051 libcoreclr.so`CLREventBase::WaitEx(unsigned int, WaitMode, PendingSync*) at synch.cpp:404
    frame #8: 0x00007f28a0270fe7 libcoreclr.so`CLREventBase::WaitEx(this=<unavailable>, dwMilliseconds=10, mode=<unavailable>, syncState=0x0000000000000000) at synch.cpp:471
    frame #9: 0x00007f28a0275263 libcoreclr.so`ThreadSuspend::SuspendRuntime(reason=<unavailable>) at threadsuspend.cpp:4187
    frame #10: 0x00007f28a02768fe libcoreclr.so`ThreadSuspend::SuspendEE(reason=<unavailable>) at threadsuspend.cpp:6512
    frame #11: 0x00007f28a018838c libcoreclr.so`ReJitManager::UpdateActiveILVersions(cFunctions=2, rgModuleIDs=0x0000559ea2e8d840, rgMethodDefs=<unavailable>, rgHrStatuses=<unavailable>, fIsRevert=240, flags=<unavailable>) at rejit.cpp:684
    frame #12: 0x00007f28a023f23c libcoreclr.so`ProfToEEInterfaceImpl::RequestReJIT(this=<unavailable>, cFunctions=2, moduleIds=0x0000559ea2e8d840, methodIds=0x0000559ea2e8eaa0) at proftoeeinterfaceimpl.cpp:8741
    frame #13: 0x00007f289c9ee613 Datadog.Trace.ClrProfiler.Native.so`trace::CorProfiler::CallTarget_RequestRejitForModule(unsigned long, trace::ModuleMetadata*, std::vector<trace::IntegrationMethod, std::allocator<trace::IntegrationMethod> > const&) + 4563
    frame #14: 0x00007f289c9ed248 Datadog.Trace.ClrProfiler.Native.so`trace::CorProfiler::ModuleLoadFinished(unsigned long, int) + 5544
    frame #15: 0x00007f28a01f7a79 libcoreclr.so`EEToProfInterfaceImpl::ModuleLoadFinished(this=0x0000559ea2542310, moduleId=139810444038688, hrStatus=0) at eetoprofinterfaceimpl.cpp:3746
[...]
```

The thread was calling `ProfToEEInterfaceImpl::RequestReJIT` which in turn was calling `ThreadSuspend::SuspendRuntime`. From there I had the full picture: the first thread was holding the global lock and indirectly calling `ThreadSuspend::SuspendRuntime`. This method blocks until all the managed threads are suspended. At the same time, the other thread was waiting on the global lock to JIT a method, which causes the deadlock.

The fix was simply to run our ReJIT operations on a dedicated native thread, outside of the global shared lock.

Interestingly, [this behavior and the recommended fix are actually documented](https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilerinfo4-requestrejit-method?WT.mc_id=DT-MVP-5003493#remarks), so I guess we should just have spent more time reading the documentation.
