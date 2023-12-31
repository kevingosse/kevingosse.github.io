---
url: writing-a-net-profiler-in-c-part-4-c54df903b9ce
canonical_url: https://medium.com/@kevingosse/writing-a-net-profiler-in-c-part-4-c54df903b9ce
title: Writing a .NET profiler in C# — Part 4
subtitle: Using NativeAOT to write a .NET profiler in C#, learning many things about
  native interop in the process.
summary: Part 4 of the series about using NativeAOT to write a .NET profiler in C#, learning many things about native interop in the process. In this part, we learn how to call methods from `ICorProfilerInfo`.
date: 2023-06-10
description: ""
tags:
- dotnet
- csharp
- nativeaot
- profiler
- source-generator
author: Kevin Gosse
thumbnailImage: /images/writing-a-net-profiler-in-c-part-4-c54df903b9ce-thumbnail.png
---

[In part 1](/writing-a-net-profiler-in-c-part-1-d3978aae9b12), we saw how NativeAOT can allow us to write a profiler in C#, and how to expose a fake COM object to use the profiling API. [In part 2](/writing-a-net-profiler-in-c-part-2-8039da001e43), we refined the solution to use instance methods instead of static methods. [In part 3](/writing-a-net-profiler-in-c-part-3-7d2c59fc017f), we automated the process using a source generator. At this point, we have everything we need to expose an instance of [`ICorProfilerCallback`](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilercallback-interface?WT.mc_id=DT-MVP-5003493). However, to write a profiler we also need to be able to call methods from [`ICorProfilerInfo`](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilerinfo-interface?WT.mc_id=DT-MVP-5003493), this will be the subject of this part.

As a reminder, we ended up with this implementation of `ICorProfilerCallback`:

```csharp
public unsafe class CorProfilerCallback2 : ICorProfilerCallback2
{
    private static readonly Guid ICorProfilerCallback2Guid = Guid.Parse("8a8cc829-ccf2-49fe-bbae-0f022228071a");

    private readonly NativeObjects.ICorProfilerCallback2 _corProfilerCallback2;

    public CorProfilerCallback2()
    {
        _corProfilerCallback2 = NativeObjects.ICorProfilerCallback2.Wrap(this);
    }

    public IntPtr Object => _corProfilerCallback2;

    public HResult Initialize(IntPtr pICorProfilerInfoUnk)
    {
        Console.WriteLine("[Profiler] ICorProfilerCallback2 - Initialize");

        // TODO: To be implemented

        return HResult.S_OK;
    }

    public HResult QueryInterface(in Guid guid, out IntPtr ptr)
    {
        if (guid == ICorProfilerCallback2Guid)
        {
            Console.WriteLine("[Profiler] ICorProfilerCallback2 - QueryInterface");

            ptr = Object;
            return HResult.S_OK;
        }

        ptr = IntPtr.Zero;
        return HResult.E_NOTIMPL;
    }

    // Stripped for brevity: the default implementation of all 70+ methods of the interface
}
```

When `Initialize` is called, we receive an instance of [IUnknown](https://learn.microsoft.com/en-us/windows/win32/api/unknwn/nn-unknwn-iunknown?WT.mc_id=DT-MVP-5003493). We need to call the `QueryInterface` on it to retrieve an instance of `ICorProfilerInfo`.

To expose objects to native code, we've seen how to create a fake vtable. To consume native objects, it's the opposite: we need to read their vtable to get the address of the methods, then invoke them.

Let's write a wrapper to invoke the methods from an instance of `IUnknown`. Because virtual objects store the address of their vtable as their first field, we just need to read a pointer at the object's location to get that vtable. We extract that logic into a property of our wrapper, for convenience:

```csharp
public unsafe struct Unknown
{
    private readonly IntPtr _self;

    public Unknown(IntPtr self)
    {
        _self = self;
    }

    private IntPtr* VTable => (IntPtr*)*(IntPtr*)_self;

    // TODO: Implement QueryInterface/AddRef/Release
}
```

Note that we declared that wrapper as struct because it doesn't need any state. In the end, it's just a fancy pointer with some embedded logic.

To invoke the methods, we retrieve their address from the appropriate slot of the vtable and cast them to [function pointers](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-9.0/function-pointers?WT.mc_id=DT-MVP-5003493). Then we just have to invoke them, making sure to pass the address of the object as first parameter because they're instance methods:

```csharp
public HResult QueryInterface(in Guid guid, out IntPtr ptr)
{
    var func = (delegate* unmanaged<IntPtr, in Guid, out IntPtr, HResult>)(*VTable);

    return func(_self, in guid, out ptr);
}

public int AddRef()
{
    var func = (delegate* unmanaged<IntPtr, int>)(*(VTable + 1));

    return func(_self);
}

public int Release()
{
    var func = (delegate* unmanaged<IntPtr, int>)(*(VTable + 2));

    return func(_self);
}
```

Our wrapper can be directly used in `ICorProfilerCallback.Initialize` to retrieve the instance of `ICorProfilerInfo`:

```csharp
public HResult Initialize(IntPtr pICorProfilerInfoUnk)
{
    Console.WriteLine("[Profiler] ICorProfilerCallback2 - Initialize");

    var iCorProfilerInfo3Guid = Guid.Parse("B555ED4F-452A-4E54-8B39-B5360BAD32A0");

    var unknown = new Unknown(pICorProfilerInfoUnk);

    var result = unknown.QueryInterface(iCorProfilerInfo3Guid, out var ptr);

    if (result == HResult.S_OK)
    {
        Console.WriteLine($"[Profiler] Successfully retrieved an instance of ICorProfilerInfo3: {ptr:x2}");
    }
    else
    {
        Console.WriteLine($"[Profiler] Failed with error code: {result:x2}");
    }

    return HResult.S_OK;
}
```

To actually use our instance of `ICorProfilerInfo`, we need to write the same kind of wrapper. However, since the interface declares tens of methods, we won't do it by hand and instead we're going to extend the source generator that we wrote [in part 3](https://minidump.net/writing-a-net-profiler-in-c-part-3-7d2c59fc017f).

Our source generator will fill the following template:

```csharp
      public unsafe struct {invokerName}
      {
          private readonly IntPtr _self;

          public {invokerName}(IntPtr self)
          {
              _self = self;
          }

          private IntPtr* VTable => (IntPtr*)*(IntPtr*)_self;

          {invokerFunctions}
      }
```

We're implementing all of this in the `EmitStubForInterface(GeneratorExecutionContext context, INamedTypeSymbol symbol)` method described [in the previous article](https://minidump.net/writing-a-net-profiler-in-c-part-3-7d2c59fc017f).

For the name of our wrapper, we just use the name of the symbol and append a suffix:

```csharp
var invokerName = $"{symbol.Name}Invoker";
```

Then we need to fill the list of functions. We declare a `StringBuilder` and start iterating on all functions from the target interface and its parents:

```csharp
var invokerFunctions = new StringBuilder();

var interfaceList = symbol.AllInterfaces.ToList();
interfaceList.Reverse();
interfaceList.Add(symbol);

foreach (var @interface in interfaceList)
{
    foreach (var member in @interface.GetMembers())
    {
        if (member is not IMethodSymbol method)
        {
            continue;
        }

        // TODO
    }
}
```

For each of the method, we start by emitting the signature:

```csharp
invokerFunctions.Append($"public {method.ReturnType} {method.Name}(");

for (int i = 0; i < method.Parameters.Length; i++)
{
    if (i > 0)
    {
        invokerFunctions.Append(", ");
    }

    var refKind = method.Parameters[i].RefKind;

    switch (refKind)
    {
        case RefKind.In:
            invokerFunctions.Append("in ");
            break;
        case RefKind.Out:
            invokerFunctions.Append("out ");
            break;
        case RefKind.Ref:
            invokerFunctions.Append("ref ");
            break;
    }

    invokerFunctions.Append($"{method.Parameters[i].Type} a{i}");
}

invokerFunctions.AppendLine(")");
```

Note that all the parameters are renamed to `a1`, `a2`, `a3`, …, to avoid any potential conflict if the arguments of the original methods have weird names.

Now we can generate the body of the method, where we fetch the address of the method from the vtable and call it with the expected arguments:

```csharp
invokerFunctions.AppendLine("{");
invokerFunctions.Append("var func = (delegate* unmanaged[Stdcall]<IntPtr");

for (int i = 0; i < method.Parameters.Length; i++)
{
    invokerFunctions.Append(", ");

    var refKind = method.Parameters[i].RefKind;

    switch (refKind)
    {
        case RefKind.In:
            invokerFunctions.Append("in ");
            break;
        case RefKind.Out:
            invokerFunctions.Append("out ");
            break;
        case RefKind.Ref:
            invokerFunctions.Append("ref ");
            break;
    }

    invokerFunctions.Append(method.Parameters[i].Type);
}

invokerFunctions.AppendLine($", {method.ReturnType}>)*(VTable + {delegateCount});");

if (method.ReturnType.SpecialType != SpecialType.System_Void)
{
    invokerFunctions.Append("return ");
}

invokerFunctions.Append("func(_self");

for (int i = 0; i < method.Parameters.Length; i++)
{
    invokerFunctions.Append($", ");

    var refKind = method.Parameters[i].RefKind;

    switch (refKind)
    {
        case RefKind.In:
            invokerFunctions.Append("in ");
            break;
        case RefKind.Out:
            invokerFunctions.Append("out ");
            break;
        case RefKind.Ref:
            invokerFunctions.Append("ref ");
            break;
    }

    invokerFunctions.Append($"a{i}");
}

invokerFunctions.AppendLine(");");
invokerFunctions.AppendLine("}");
```

That's a lot of code, but it's mostly enumerating the arguments to generate the method call, and some special casing in case the method returns `void`.

Last but not least, we replace the placeholders in our template:

```csharp
sourceBuilder.Replace("{invokerFunctions}", invokerFunctions.ToString());
sourceBuilder.Replace("{invokerName}", invokerName);
```

With that, we can go back to our implementation of `ICorProfilerCallback.Initialize` and replace `Unknown` by our automatically generated implementation:

```csharp
  public HResult Initialize(IntPtr pICorProfilerInfoUnk)
  {
      Console.WriteLine("[Profiler] ICorProfilerCallback2 - Initialize");

      var iCorProfilerInfo3Guid = Guid.Parse("B555ED4F-452A-4E54-8B39-B5360BAD32A0");

      var unknown = new NativeObjects.IUnknownInvoker(pICorProfilerInfoUnk);

      var result = unknown.QueryInterface(iCorProfilerInfo3Guid, out var ptr);

      if (result == HResult.S_OK)
      {
          Console.WriteLine($"[Profiler] Successfully retrieved an instance of ICorProfilerInfo3: {ptr:x2}");

          var corProfilerInfo = new NativeObjects.ICorProfilerInfo3Invoker(ptr);
          // Can start interacting with ICorProfilerInfo
      }
      else
      {
          Console.WriteLine($"[Profiler] Failed with error code: {result:x2}");
      }

      return HResult.S_OK;
  }
```

With this, we finally have all the pieces of the puzzle to actually start writing a profiler.

{{<image classes="fancybox center" src="/images/writing-a-net-profiler-in-c-part-4-c54df903b9ce-1.webp" >}}

As a reminder, all the code [is available on GitHub](https://github.com/kevingosse/ManagedDotnetProfiler).
