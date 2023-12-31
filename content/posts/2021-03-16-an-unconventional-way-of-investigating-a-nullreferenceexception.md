---
url: an-unconventional-way-of-investigating-a-nullreferenceexception-5628cca01d6a
canonical_url: https://medium.com/@kevingosse/an-unconventional-way-of-investigating-a-nullreferenceexception-5628cca01d6a
title: An unconventional way of investigating a NullReferenceException
subtitle: Digging into a bug in the .NET ARM64 runtime, learning about dispatch stubs,
  and using that knowledge to diagnose a NullReferenceException
date: 2021-03-16
description: ""
tags:
- csharp
- dotnet
- dotnet-core
- programming
- debugging
author: Kevin Gosse
---

# The crash

This one started when trying to understand why an integration test was failing, only on Linux with ARM64.

As I had no ARM64 dev environment available, I first tried adding more and more traces and let the test run in the CI, without much success.

![][image_ref_MCo0b29MQ3JVM1FoMXJ1dW04]

Eventually, I realized this was leading nowhere, and took the time to setup an ARM64 VM to investigate further. After running the test with LLDB (see [my previous article](/investigating-a-native-deadlock-in-a-net-linux-application-97979a005ebd) to learn how to fetch the symbols for the CLR), I found out that the process was raising two segmentations faults, and the second one caused the crash:

```
* thread #1, name = 'dotnet', stop reason = signal SIGSEGV
  * frame #0: 0x0000ffffb76ccc40 libcoreclr.so`AdjustContextForVirtualStub(pExceptionRecord=0x0000000000000000, pContext=0x0000ffff077f38e0) at stubs.cpp:1173:40
    frame #1: 0x0000ffffb76d4a18 libcoreclr.so`UnwindManagedExceptionPass1(ex=<unavailable>, frameContext=<unavailable>) at exceptionhandling.cpp:4579:17
    frame #2: 0x0000ffffb76d4c08 libcoreclr.so`DispatchManagedException(ex=0x0000ffff077f3cd0, isHardwareException=<unavailable>) at exceptionhandling.cpp:4686:17
    frame #3: 0x0000ffffb763c0cc libcoreclr.so`IL_Throw(obj=<unavailable>) at jithelpers.cpp:4195:5
[managed frames]
```

[I opened an issue](https://github.com/dotnet/runtime/issues/49070) on the dotnet/runtime repository, and David Mason was quick to track it down to a missing null check in `AdjustContextForVirtualStub`.

It turns out that the assertion ".NET checks for nullity when doing a virtual call" is not exactly true. In some situations, when checking the type of an object, the runtime assumes the value is not null, then catches the access violation/segmentation fault when trying to dereference the instance. The fault is then converted to a `NullReferenceException`. The end result is the same as if .NET was explicitly checking for nullity. So what happened in my test application was:

* I was trying to call a virtual method on a null reference

* This caused a segmentation fault, which was caught by the runtime and converted into a `NullReferenceException`. An important point to understand is that the fault/exception occurred during the dispatch of the virtual call, **which is not considered as managed code**.

* The exception was rethrown in the catch block of the method

* When unwinding the stack, it hit a special case in `UnwindManagedExceptionPass1` when the exception originates from native code. That code path was missing a null check and caused the fatal segmentation fault.

I built a custom version of the CLR with the additional null check, and as predicted this fixed the crash. End of story?

# Writing a repro

The story could have ended here, but I felt like I was still missing something to get the full picture. .NET isn't widely used on ARM64, but I figured out that if the issue was as simple as "crashes when invoking a virtual method on a null reference", the bug would have been found much sooner.

To understand the exact conditions for the crash, I decided to try and write a repro. I started with a simple virtual call on a null reference:

```csharp
public class Program
{
    static void Main(string[] args)
    {
        try
        {
            Request();
        }
        catch (Exception ex)
        {
            Console.WriteLine("Caught: " + ex);
        }
    }

    public static void Request()
    {
        try
        {
            var response = GetValue();
            _ = response.GetHashCode(); // Virtual call on null reference
        }
        catch (Exception ex)
        {
            throw new Exception("Rethrowing", ex);
        }
    }

    [MethodImpl(MethodImplOptions.NoInlining)] // Prevent any kind of unexpected compiler optimization
    public static object GetValue()
    {
        return null;
    }
}
```

Without much surprise, this program didn't crash. So there was more to the problem than just "a virtual call on a null reference".

When running the repro program with LLDB, it broke on a segmentation fault:

```
(lldb) run
Process 111393 launched: '/home/ubuntu/testconsole-blog/bin/Release/net5.0/testconsole' (aarch64)
Process 111393 stopped
* thread #1, name = 'testconsole', stop reason = signal SIGSEGV: invalid address (fault address: 0x0)
    frame #0: 0x0000ffff7e42af88
->  0xffff7e42af88: ldr    x1, [x1]
    0xffff7e42af8c: ldr    x1, [x1, #0x40]
    0xffff7e42af90: ldr    x1, [x1, #0x18]
    0xffff7e42af94: blr    x1
(lldb) ip2md 0x0000ffff7e42af88
MethodDesc:   0000ffff7e4b1f90
Method Name:          testconsole.Program.Request()
[...]
```

The segmentation fault occurred in the `Request` method, which was expected. But if I tried the same thing in my crashing app, the segfault would happen in a non-managed method:

```
Process 64731 stopped
* thread #16, name = '.NET ThreadPool', stop reason = signal SIGSEGV: invalid address (fault address: 0x0)
    frame #0: 0x0000ffff7d866300
->  0xffff7d866300: ldr    x13, [x0]
    0xffff7d866304: adr    x9, #0x1c
    0xffff7d866308: ldp    x10, x12, [x9]
    0xffff7d86630c: cmp    x13, x10

(lldb) ip2md 0x0000ffff7d866300
Failed to request MethodData, not in JIT code range
IP2MD 0xffff7d866300  failed
```

What was this method? It wasn't exported in the .NET CLR symbols, yet it wasn't a managed method either. To get more information, I enabled the perf map generation ([by setting the `COMPlus_PerfMapEnable` environment variable](https://docs.microsoft.com/en-us/dotnet/core/run-time-config/debugging-profiling?WT.mc_id=DT-MVP-5003493#write-perf-map)). The perf map is a simple text file in which the JIT stores the name and address of the methods it compiles. Luckily, I found the address of the mysterious method in there, with the name `GenerateDispatchStub<GenerateDispatchStub>`.

I then looked into the code of the CLR to understand [what this name was associated to](https://github.com/dotnet/runtime/blob/v5.0.3/src/coreclr/src/vm/virtualcallstub.cpp#L2784).

The CLR was creating a `DispatchHolder`:

```c++
DispatchHolder * holder = (DispatchHolder*) (void*)
    dispatch_heap->AllocAlignedMem(dispatchHolderSize, CODE_SIZE_ALIGN);

```

Then calling `Initialize` to emit some code:

```c++
holder->Initialize(addrOfCode,
                   addrOfFail,
                   (size_t)pMTExpected
```

That method had [a specific implementation for ARM64](https://github.com/dotnet/runtime/blob/v5.0.3/src/coreclr/src/vm/arm64/virtualcallstubcpu.hpp#L109):

```c++
void  Initialize(PCODE implTarget, PCODE failTarget, size_t expectedMT)
{
    // ldr x13, [x0] ; methodTable from object in x0
    // adr x9, _expectedMT ; _expectedMT is at offset 28 from pc
    // ldp x10, x12, [x9] ; x10 = _expectedMT & x12 = _implTarget
    // cmp x13, x10
    // bne failLabel
    // br x12
    // failLabel
    // ldr x9, _failTarget ; _failTarget is at offset 24 from pc
    // br x9
    // _expectedMT
    // _implTarget
    // _failTarget

    _stub._entryPoint[0] = DISPATCH_STUB_FIRST_DWORD; // 0xf940000d
    _stub._entryPoint[1] = 0x100000e9;
    _stub._entryPoint[2] = 0xa940312a;
    _stub._entryPoint[3] = 0xeb0a01bf;
    _stub._entryPoint[4] = 0x54000041;
    _stub._entryPoint[5] = 0xd61f0180;
    _stub._entryPoint[6] = 0x580000c9;
    _stub._entryPoint[7] = 0xd61f0120;

    _stub._expectedMT = expectedMT;
    _stub._implTarget = implTarget;
    _stub._failTarget = failTarget;
}
```

The instructions in the comment matched exactly what LLDB was showing me:

```
->  0xffff7d866300: ldr    x13, [x0]
    0xffff7d866304: adr    x9, #0x1c
    0xffff7d866308: ldp    x10, x12, [x9]
    0xffff7d86630c: cmp    x13, x10
```

But this was different from what I was getting in my repro app:

```
->  0xffff7e42af88: ldr    x1, [x1]
    0xffff7e42af8c: ldr    x1, [x1, #0x40]
    0xffff7e42af90: ldr    x1, [x1, #0x18]
    0xffff7e42af94: blr    x1
```

So it seemed like the crashing app was using a "dispatch stub", but my repro app did not. Did it matter?

Looking back [at the method in which the crash happened](https://github.com/dotnet/runtime/blob/v5.0.3/src/coreclr/src/vm/arm64/stubs.cpp#L1126):

```c++
BOOL
AdjustContextForVirtualStub(
        EXCEPTION_RECORD *pExceptionRecord,
        CONTEXT *pContext)
{
    LIMITED_METHOD_CONTRACT;

    Thread * pThread = GetThread();

    // We may not have a managed thread object. Example is an AV on the helper thread.
    // (perhaps during StubManager::IsStub)
    if (pThread == NULL)
    {
        return FALSE;
    }

    PCODE f_IP = GetIP(pContext);

    VirtualCallStubManager::StubKind sk;
    VirtualCallStubManager::FindStubManager(f_IP, &sk);

    if (sk == VirtualCallStubManager::SK_DISPATCH)
    {
        if (*PTR_DWORD(f_IP) != DISPATCH_STUB_FIRST_DWORD)
        {
            _ASSERTE(!"AV in DispatchStub at unknown instruction");
            return FALSE;
        }
    }
    else
    if (sk == VirtualCallStubManager::SK_RESOLVE)
    {
        if (*PTR_DWORD(f_IP) != RESOLVE_STUB_FIRST_DWORD)
        {
            _ASSERTE(!"AV in ResolveStub at unknown instruction");
            return FALSE;
        }
    }
    else
    {
        return FALSE;
    }

    PCODE callsite = GetAdjustedCallAddress(GetLR(pContext));

    // Lr must already have been saved before calling so it should not be necessary to restore Lr

    pExceptionRecord->ExceptionAddress = (PVOID)callsite;
    SetIP(pContext, callsite);

    return TRUE;
}
```

The crashed occurred because `pExceptionRecord` was null, line 48. If my repro app didn't crash, it either meant that the method wasn't called, `pExceptionRecord` wasn't null, or the method exited earlier. I confirmed by setting a breakpoint that the method was called with a null argument. So it would mean that either `pThread` was null (which seemed incredibly unlikely), or the return value of `VirtualCallStubManager::FindStubManager` was different from `SK_DISPATCH` or `SK_LOOKUP`. "`SK_DISPATCH`" ? Whatever that was, it seemed consistent with the dispatch stub mentioned earlier.

# Learning about stubs

While looking for more information about what those stubs were, I ended up in the [Book of The Runtime](https://github.com/dotnet/coreclr/blob/master/Documentation/botr/virtual-stub-dispatch.md#dispatch-stubs). There's a lot to digest in there, but the bottom-line is that whenever a method is called on an interface (and only an interface), a special resolution mechanism is used, named "virtual stub dispatch". That resolution mechanism uses 3 types of stubs: lookup stubs, resolve stubs, or the dispatch stub we were looking for.

* The lookup stub just calls the resolver with the right parameters to resolve the address of the target method.

* The dispatch stub is an optimistic dispatch mechanism: it's hardcoded with the expected implementation type of the interface and the corresponding address of the method. When invoked, it performs a quick type check and jumps to the address. If the type check fails, it instead fallbacks to the resolve stub.

* The resolve stub is pretty much a cache. If checks if the address of the target method is already in the cache. If yes, it jumps to it. If not, it calls the resolver and adds the new entry to the cache.

With that information in hand, I modified my repro to use interfaces:

```csharp
public class Program
{
    public interface IClient
    {
        object GetResponse();
    }

    static void Main(string[] args)
    {
        try
        {
            Request();
        }
        catch (Exception ex)
        {
            Console.WriteLine("Caught: " + ex);
        }
    }

    public static void Request()
    {
        try
        {
            var client = GetClient();
            _ = client.GetResponse(); // Virtual call on null reference
        }
        catch (Exception ex)
        {
            throw new Exception("Rethrowing", ex);
        }
    }

    [MethodImpl(MethodImplOptions.NoInlining)] // Prevent any kind of unexpected compiler optimization
    public static IClient GetClient()
    {
        return null;
    }
}

```

But it still wouldn't crash. Worse, it wouldn't even cause a segmentation fault anymore!

After reading a bit more about virtual dispatch stubs, it turns out that the type of stub used for a given call site changes through the life of a process. When compiling a method, the JIT emits a lookup stub at the call site. When invoked, that stub will emit both a dispatch stub and a resolve stub, and will backpatch the call site to use the new stubs. The reason the JIT does not directly emit a dispatch/resolve stub is apparently because it's missing some contextual information, that is only available at invocation time.

In my repro app, since I was calling the method only once, a lookup stub was used. I needed to call the method multiple times to make sure the dispatch stub was emitted, and then cause the `NullReferenceException`:

```csharp
public class Program
{
    public interface IClient
    {
        object GetResponse();
    }

    public class Client : IClient
    {
        public object GetResponse() => new object();
    }

    static void Main(string[] args)
    {
        try
        {
            Request(false);
            Request(false);
            Request(true);
        }
        catch (Exception ex)
        {
            Console.WriteLine("Caught: " + ex);
        }
    }

    public static void Request(bool isNull)
    {
        try
        {
            var client = GetClient(isNull);
            _ = client.GetResponse(); // Virtual call on null reference
        }
        catch (Exception ex)
        {
            throw new Exception("Rethrowing", ex);
        }
    }

    [MethodImpl(MethodImplOptions.NoInlining)] // Prevent any kind of unexpected compiler optimization
    public static IClient GetClient(bool isNull)
    {
        return isNull ? null : new Client();
    }
}

```

And this time it had the expected result:

```bash
$ ./bin/Release/net5.0/testconsole
Segmentation fault
```

There was still one thing puzzling me: I needed to call `Request(false)` two times for the dispatch stub to be emitted. I was expecting the JIT to emit a lookup stub during compilation, then the lookup stub to emit a dispatch stub during the first invocation. So only one `Request(false)` would have been needed. Why did I need two?

I found the answer [in a comment in the source code of the resolver](https://github.com/dotnet/runtime/blob/v5.0.3/src/coreclr/src/vm/virtualcallstub.cpp#L1539) (the resolver is the bit of code called by the lookup stub):

> Note, if we encounter a method that hasn't been jitted yet, we will return the prestub, which should cause it to be jitted and we will be able to build the dispatching stub on a later call thru the call site

Therefore, the sequence of events was:

* `Request` is compiled by the JIT. At this point, contextual information is missing and a lookup stub is emitted.

* First call to `Request(false)` : when calling `IClient.GetResponse`, the lookup stub is invoked. It resolves the call to `Client.GetResponse` but notices that this method hasn't been JITted. Since it doesn't know the final location of the code, if just returns the address of the prestub. When the prestub is executed, it triggers the JIT compilation of the method.

* Second call to `Request(false)` : when calling `IClient.GetResponse`, the lookup stub is invoked. It resolves the call to `Client.GetResponse` and emits a dispatch stub that points to the address of the JITted code.

* Call to `Request(true)` : the dispatch stub is used for resolution, but the instance is null. This causes a segmentation fault inside of the stub, which in turn will cause the crash when the stack is unwind.

If my analysis was correct, I would need only one call to `Request(false)` if I made sure `Client.GetResponse` was already JIT-compiled at the moment of the invocation. I confirmed that by making this change to the repro:

```csharp
    static void Main(string[] args)
    {
        try
        {
            _ = new Client().GetResponse(); // Force the JIT compilation
            Request(false); // Only one Request(false) should be needed now
            Request(true);
        }
        catch (Exception ex)
        {
            Console.WriteLine("Caught: " + ex);
        }
    }
```

And indeed, it crashed with a segmentation fault.

# Back to my NullReferenceException

Meanwhile, even though it wasn't causing a crash anymore with the patched CLR, my integration test was still throwing a `NullReferenceException` somewhere.

In normal conditions, debugging a null reference is hardly something noteworthy. But for various reasons, such as the remote ARM64 VM or the fact that the issue happened only with our profiler attached, I couldn't attach the Visual Studio debugger and it was very difficult for me to make any change in the code. All I knew was that the error occurred in this method from the Elasticsearch.net client:

```csharp
  public TResponse Request<TResponse>(HttpMethod method, string path, PostData data = null, IRequestParameters requestParameters = null)
    where TResponse : class, IElasticsearchResponse, new()
  {
    using (var pipeline = PipelineProvider.Create(Settings, DateTimeProvider, MemoryStreamFactory, requestParameters))
    {
      pipeline.FirstPoolUsage(Settings.BootstrapLock);

      var requestData = new RequestData(method, path, data, Settings, requestParameters, MemoryStreamFactory);
      Settings.OnRequestDataCreated?.Invoke(requestData);
      TResponse response = null;

      var seenExceptions = new List<PipelineException>();
      foreach (var node in pipeline.NextNode())
      {
        requestData.Node = node;
        try
        {
          pipeline.SniffOnStaleCluster();
          Ping(pipeline, node);
          response = pipeline.CallElasticsearch<TResponse>(requestData);
          if (!response.ApiCall.SuccessOrKnownError)
          {
            pipeline.MarkDead(node);
            pipeline.SniffOnConnectionFailure();
          }
        }
        catch (PipelineException pipelineException) when (!pipelineException.Recoverable)
        {
          HandlePipelineException(ref response, pipelineException, pipeline, node, seenExceptions);
          break;
        }
        catch (PipelineException pipelineException)
        {
          HandlePipelineException(ref response, pipelineException, pipeline, node, seenExceptions);
        }
        catch (Exception killerException)
        {
          throw new UnexpectedElasticsearchClientException(killerException, seenExceptions)
          {
            Request = requestData,
            Response = response?.ApiCall,
            AuditTrail = pipeline?.AuditTrail
          };
        }
        if (response == null || !response.ApiCall.SuccessOrKnownError) continue;

        pipeline.MarkAlive(node);
        break;
      }
      return FinalizeResponse(requestData, pipeline, seenExceptions, response);
    }
  }
```

There was a lot of things in there that could be null. And the fact that it came from an external library made things even harder. I needed to figure a way to debug that issue from LLDB without any code change.

So I ran the application again with LLDB, until the location of the first segmentation fault. This time, thanks to everything I learned above, I knew the segmentation fault was occurring in the dispatch stub, with the now familiar `ldr x13, [x0]` instruction:

```
Process 64731 stopped
* thread #16, name = '.NET ThreadPool', stop reason = signal SIGSEGV: invalid address (fault address: 0x0)
    frame #0: 0x0000ffff7d866300
->  0xffff7d866300: ldr    x13, [x0]
    0xffff7d866304: adr    x9, #0x1c
    0xffff7d866308: ldp    x10, x12, [x9]
    0xffff7d86630c: cmp    x13, x10
```

The [comment in the stub implementation](https://github.com/dotnet/runtime/blob/v5.0.3/src/coreclr/src/vm/arm64/virtualcallstubcpu.hpp#L111-L113) indicates:

```c++
        // ldr x13, [x0] ; methodTable from object in x0
        // adr x9, _expectedMT ; _expectedMT is at offset 28 from pc
        // ldp x10, x12, [x9] ; x10 = _expectedMT & x12 = _implTarget
```

So by decompiling the stub, I should be able to retrieve the hardcoded `_expectedMT` and `_implTarget`:

```
(lldb) disassemble -s 0x0000ffff7d866300 -c 30
->  0xffff7d866300: ldr    x13, [x0]
    0xffff7d866304: adr    x9, #0x1c
    0xffff7d866308: ldp    x10, x12, [x9]
    0xffff7d86630c: cmp    x13, x10
    0xffff7d866310: b.ne   0xffff7d866318
    0xffff7d866314: br     x12
    0xffff7d866318: ldr    x9, #0x18
    0xffff7d86631c: br     x9
    0xffff7d866320: .long  0x7fdd0f50                ; unknown opcode
    0xffff7d866324: udf    #0xffff
    0xffff7d866328: .long  0x80be2a98                ; unknown opcode
    0xffff7d86632c: udf    #0xffff
    0xffff7d866330: .long  0x7d8be114                ; unknown opcode
    0xffff7d866334: udf    #0xffff
```

We can see the `ldp x10, x12, [x9]` instruction. According to the comments, it loads the `_expectedMT` and `_implTarget` values we're looking for. `ldp` is an ARM instruction that loads two words from the target address (stored in `x9`) and stores them in the designated registers (`x10` and `x12` ). The value of `x9` was set by the instruction `adr x9, #0x1c`. That means "get the address of the current instruction, add the offset `0x1c` , and stores it in `x9`". The address of that instruction is `0xffff7d866304`, so it stores the value `0xffff7d866304 + 0x1c = 0xffff7d866320` in the register. At `0xffff7d866320`, we can see a sequence of values:

```
0x7fdd0f50
0xffff
0x80be2a98
0xffff
```

Stitched together, it means that the instruction `ldp x10, x12, [x9]` stores `0xffff7dd0f50` into `x10` and `0xffff80be2a98` into `x12`. At that point I had all the information I needed! I could then inspect the expected MT, and from there see what method was stored at address `0xffff80be2a98`:

```
(lldb) dumpmt -md 0xffff7fdd0f50
EEClass:         0000FFFF7FDC95D0
Module:          0000FFFF7E5838E8
Name:            Nest.CreateIndexResponse
mdToken:         0000000002000A4D
File:            /home/ubuntu/git/dd-trace-dotnet/test/test-applications/integrations/Samples.Elasticsearch/bin/Debug/net5.0/publish/Nest.dll
BaseSize:        0x38
ComponentSize:   0x0
DynamicStatics:  false
ContainsPointers true
Slots in VTable: 17
Number of IFaces in IFaceMap: 4
--------------------------------------
MethodDesc Table
           Entry       MethodDesc    JIT Name
0000FFFF7E4100C0 0000FFFF7E400B90    JIT System.Object.Finalize()
0000FFFF80CBF3F0 0000FFFF7E795928    JIT Nest.ResponseBase.ToString()
[...]
0000FFFF80BE2A98 0000FFFF7E795808    JIT Nest.ResponseBase.Elasticsearch.Net.IElasticsearchResponse.get_ApiCall()
[...]
```

From there, I knew the `NullReferenceException` occurred when trying to call the getter of the `ApiCall` property on an instance of `Nest.CreateIndexResponse`. With that information, finding the exact line of failure in the source code of the method was trivial:

```csharp
response = pipeline.CallElasticsearch<TResponse>(requestData);
if (!response.ApiCall.SuccessOrKnownError)
{
  pipeline.MarkDead(node);
  pipeline.SniffOnConnectionFailure();
}
```

Thanks to that, I was able to understand what caused that value to be null and fix the issue. And finally close the pull request.

![][image_ref_MSpMQmNmaGdhVVJhZ2JYRGZjTGtIMmJRLnBuZw==]


[image_ref_MCo0b29MQ3JVM1FoMXJ1dW04]: data:image/png;base64,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wDmUrBQREZHLlUKPTDaTbs3PZLTpH77PnXcv40DLIXr7+3A73cQSFnveaeK119/EmzeVgVCMYCiK159HJJ4gYVvYSQsXDlxON07TTcJ08c6BQ+TNX8pNK9aku1vnoYiZs4D2nTSMmj8boH5zqhb/HSp3LSIiIiITTNPeLqEXfvFLvvPYN7GiNqFYkEQiholFQX4+bpeHQFcv3QMDhAjhc2aR4/HjsB0kbRuP103+lFz+bsMG7nvg/nR35YKMu+bnA3dCFhERERG5OBR+LrHBgQGerfkpv3ruF7QFjgEW0VCYRMLCdpr0h8I43Ra5WT6ybBMrkWTKtOmsWfsQax/5Gtk5OenugoiIiIjIpKTwk0a/376d39f/jsY9fybQ1s6ho61EYzFmzSzkutJrWFC2gKUfu4PyFRNVzU1EREREJHNM6vDT2X3i5M/Tp+ansSUiIiIiInK5U8EDERERERHJCAo/IiIiIiKSERR+REREREQkIyj8iIiIiIhIRlD4ERERERGRjKDwIyIiIiIiGUHhR0REREREMoLCj4iIiIiIZASFHxERERERyQgKPyIiIiIikhEUfkREREREJCMo/IiIiIiISEZQ+BERERERkYyg8CMiIiIiIhlB4UdERERERDKCwo+IiIiIiGQEhR8REREREckICj8iIiIiIpIRFH5ERERERCQjKPyIiIiIiEhGUPgREREREZGMoPAjIiIiIiIZQeFHREREREQygsKPiIiIiIhkBIUfERERERHJCAo/IiIiIiKSERR+REREREQkIyj8iIiIiIhIRlD4ERERERGRjKDwIyIiIiIiGUHhR0REREREMsL/B4yC1gvnrvyrAAAAAElFTkSuQmCC
[image_ref_MSpMQmNmaGdhVVJhZ2JYRGZjTGtIMmJRLnBuZw==]: data:image/png;base64,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