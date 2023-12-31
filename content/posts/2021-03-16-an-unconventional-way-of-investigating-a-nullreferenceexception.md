---
url: an-unconventional-way-of-investigating-a-nullreferenceexception-5628cca01d6a
canonical_url: https://medium.com/@kevingosse/an-unconventional-way-of-investigating-a-nullreferenceexception-5628cca01d6a
title: An unconventional way of investigating a NullReferenceException
subtitle: Digging into a bug in the .NET ARM64 runtime, learning about dispatch stubs,
  and using that knowledge to diagnose a NullReferenceException
date: 2021-03-16
description: ""
tags:
- dotnet
- debugging
- lldb
- windbg
- arm64
- assembly
- cpp
author: Kevin Gosse
thumbnailImage: /images/an-unconventional-way-of-investigating-a-nullreferenceexception-5628cca01d6a-1.webp
---

# The crash

This one started when trying to understand why an integration test was failing, only on Linux with ARM64.

As I had no ARM64 dev environment available, I first tried adding more and more traces and let the test run in the CI, without much success.

{{<image classes="fancybox center" src="/images/an-unconventional-way-of-investigating-a-nullreferenceexception-5628cca01d6a-1.webp" >}}

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

{{<image classes="fancybox center" src="/images/an-unconventional-way-of-investigating-a-nullreferenceexception-5628cca01d6a-2.webp" >}}
