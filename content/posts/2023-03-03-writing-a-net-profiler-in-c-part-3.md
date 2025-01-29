---
url: writing-a-net-profiler-in-c-part-3-7d2c59fc017f
canonical_url: https://medium.com/@kevingosse/writing-a-net-profiler-in-c-part-3-7d2c59fc017f
title: Writing a .NET profiler in C#   — Part 3
subtitle: Using NativeAOT to write a .NET profiler in C#, learning many things about
  native interop in the process.
summary: Part 3 of the series about using NativeAOT to write a .NET profiler in C#, learning many things about native interop in the process. In this part, we write a source generator to automatically generate the boilerplate code needed to implement the `ICorProfilerCallback` interface.
date: 2023-03-03
description: ""
tags:
- dotnet
- nativeaot
- profiler
- source-generator
author: Kevin Gosse
thumbnailImage: /images/writing-a-net-profiler-in-c-part-3-7d2c59fc017f-1.webp
---

[In part 1](/writing-a-net-profiler-in-c-part-1-d3978aae9b12), we saw how NativeAOT can allow us to write a profiler in C#, and how to expose a fake COM object to use the profiling API. [In part 2](/writing-a-net-profiler-in-c-part-2-8039da001e43), we refined the solution to use instance methods instead of static methods. Now that we know how to interact with the profiling API, we're going to write a source generator to automatically generate the boilerplate code needed to implement the 70+ methods declared in the [`ICorProfilerCallback`](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilercallback-interface?WT.mc_id=DT-MVP-5003493) interface.

First we need to manually [convert the `ICorProfilerCallback` interface to C#](https://github.com/kevingosse/ManagedDotnetProfiler/blob/a204080e9299b9bb774f0f5d291640fe081fc04c/ManagedDotnetProfiler/ICorProfilerCallback.cs). Technically it would have been possible to automatically generate this from the C++ header files, but the same C++ code can be translated in different ways in C#, and so it's important to understand the purpose of the functions to convert them with the right semantics.

To take an actual example, consider the [`JITInlining`](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilercallback-jitinlining-method?WT.mc_id=DT-MVP-5003493) function. The prototype in C++ is:

```csharp
HRESULT JITInlining(FunctionID callerId, FunctionID calleeId, BOOL *pfShouldInline);
```

A naïve conversion in C# would be:

```csharp
HResult JITInlining(FunctionId callerId, FunctionId calleeId, bool* pfShouldInline);
```

However, we don't actually need pointers here. If the `pfShouldInline` argument is read-only we could translate it to:

```csharp
HResult JITInlining(FunctionId callerId, FunctionId calleeId, in bool pfShouldInline);
```

But if we look at the documentation of the function, we understand that `pfShouldInline` is a value that should be set by the function itself. So we should instead use the `out` keyword:

```csharp
HResult JITInlining(FunctionId callerId, FunctionId calleeId, out bool pfShouldInline);
```

In other cases, we will use `in` or `ref` depending on the intent. That's why we can't completely automate the process.

After the interface has been translated to C#, we can proceed with creating the source generator. Note that I don't intend to write a state-of-the-art source generator, mainly because the API is extremely complex (yes, coming from somebody who's explaining how to write a profiler in C#), and you can check [Andrew Lock's excellent articles](https://andrewlock.net/creating-a-source-generator-part-1-creating-an-incremental-source-generator/) for that.

# Writing a source generator

To create the source generator, we add a class library project to the solution that targets `netstandard2.0`, and we add references to `Microsoft.CodeAnalysis.CSharp` and `Microsoft.CodeAnalysis.Analyzers`:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <LangVersion>latest</LangVersion>
    <IsRoslynComponent>true</IsRoslynComponent>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.0.1" PrivateAssets="all" />
    <PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.3.3">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
  </ItemGroup>

</Project>
```

Then we add a class that implements `ISourceGenerator` , and decorate it with the `[Generator]` attribute:

```csharp
[Generator]
public class NativeObjectGenerator : ISourceGenerator
{
    public void Initialize(GeneratorInitializationContext context)
    {
    }

    public void Execute(GeneratorExecutionContext context)
    {
    }
}
```

The first thing we want to do is to emit a `[NativeObject]` attribute. We will use it to decorate the interfaces that we want to run the source generator on. We use `RegisterForPostInitialization` to run this code early in the pipeline:

```csharp
[Generator]
public class NativeObjectGenerator : ISourceGenerator
{
    public void Initialize(GeneratorInitializationContext context)
    {
        context.RegisterForPostInitialization(EmitAttribute);

    }

    public void Execute(GeneratorExecutionContext context)
    {
    }

    private void EmitAttribute(GeneratorPostInitializationContext context)
    {
        context.AddSource("NativeObjectAttribute.g.cs", """
    using System;

    [AttributeUsage(AttributeTargets.Interface, Inherited = false, AllowMultiple = false)]
    internal class NativeObjectAttribute : Attribute { }
    """);
    }
}
```

Now we need to register a `ISyntaxContextReceiver` to inspect types and detect which ones are decorated with our `[NativeObject]` attribute.

```csharp
public class SyntaxReceiver : ISyntaxContextReceiver
{
    public List<INamedTypeSymbol> Interfaces { get; } = new();

    public void OnVisitSyntaxNode(GeneratorSyntaxContext context)
    {
        if (context.Node is InterfaceDeclarationSyntax classDeclarationSyntax
            && classDeclarationSyntax.AttributeLists.Count > 0)
        {
            var symbol = (INamedTypeSymbol)context.SemanticModel.GetDeclaredSymbol(classDeclarationSyntax);

            if (symbol.GetAttributes().Any(a => a.AttributeClass.ToDisplayString() == "NativeObjectAttribute"))
            {
                Interfaces.Add(symbol);
            }
        }
    }
}
```

Basically, the syntax receiver is going to be called for every node in the syntax tree. We check if that node is an interface declaration, and if it is we inspect the attributes to find `NativeObjectAttribute`. There are probably a lot of things that can be improved, especially to confirm if it's *our* `NativeObjectAttribute`, but we'll say that's good enough for our purpose.

The syntax receiver needs to be registered during the initialization of our source generator:

```csharp
public void Initialize(GeneratorInitializationContext context)
{
    context.RegisterForPostInitialization(EmitAttribute);
    context.RegisterForSyntaxNotifications(() => new SyntaxReceiver());
}
```

Finally, in the `Execute` method, we retrieve the list of interfaces that were stored in the syntax receiver, and we generate the code for it:

```csharp
public void Execute(GeneratorExecutionContext context)
{
    if (!(context.SyntaxContextReceiver is SyntaxReceiver receiver))
    {
        return;
    }

    foreach (var symbol in receiver.Interfaces)
    {
        EmitStubForInterface(context, symbol);
    }
}
```

{{<image classes="fancybox center" src="/images/writing-a-net-profiler-in-c-part-3-7d2c59fc017f-1.webp" >}}

# Generating the native wrapper

For the `EmitStubForInterface` method, we could use a template engine, but instead we will just rely on a good old `StringBuilder` and calls to `Replace`.

First, we create our template:

```csharp
var sourceBuilder = new StringBuilder("""
    using System;
    using System.Runtime.InteropServices;

    namespace NativeObjects
    {
        {visibility} unsafe class {typeName} : IDisposable
        {
            private {typeName}({interfaceName} implementation)
            {
                const int delegateCount = {delegateCount};

                var obj = (IntPtr*)NativeMemory.Alloc((nuint)2 + delegateCount, (nuint)IntPtr.Size);
    
                var vtable = obj + 2;

                *obj = (IntPtr)vtable;
    
                var handle = GCHandle.Alloc(implementation);
                *(obj + 1) = GCHandle.ToIntPtr(handle);

    {functionPointers}

                Object = (IntPtr)obj;
            }

            public IntPtr Object { get; private set; }

            public static {typeName} Wrap({interfaceName} implementation) => new(implementation);

            public static implicit operator IntPtr({typeName} stub) => stub.Object;

            ~{typeName}()
            {
                Dispose();
            }

            public void Dispose()
            {
                if (Object != IntPtr.Zero)
                {
                    NativeMemory.Free((void*)Object);
                    Object = IntPtr.Zero;
                }

                GC.SuppressFinalize(this);
            }

            private static class Exports
            {
    {exports}
            }
        }
    }
""");
```

If there are parts that you don't understand, remember to check [the previous article](/writing-a-net-profiler-in-c-part-2-8039da001e43). The only new thing here is the finalizer and the `Dispose` method, where we call `NativeMemory.Free` to free the memory allocated for that object. Then we need to fill all the templated parts: `{visibility}`, `{typeName}`, `{interfaceName}`, `{delegateCount}`, `{functionPointers}`, and `{exports}`.

First the easy ones:

```csharp
    var interfaceName = symbol.ToString();
    var typeName = $"{symbol.Name}";
    var visibility = symbol.DeclaredAccessibility.ToString().ToLower();

    // To be filled later
    int delegateCount = 0;
    var exports = new StringBuilder();
    var functionPointers = new StringBuilder();
```

For an interface `MyProfiler.ICorProfilerCallback`, we will generate a wrapper of type `NativeObjects.ICorProfilerCallback`. That's why we store the fully qualified name in `interfaceName` (= `MyProfiler.ICorProfilerCallback`), and just the type name in `typeName` (= `ICorProfilerCallback`).

Then we want to generate the list of exports and their function pointers. I want the source generator to support inheritance, to avoid code duplication as `ICorProfilerCallback13` implements `ICorProfilerCallback12`, which itself implemeents `ICorProfilerCallback11`, and so on. So we extract the list of interfaces that the target interface inherit from, and we extract the methods for each of them:

```csharp
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

        // TODO: Inspect the method
    }
}
```

For a `QueryInterface(in Guid guid, out IntPtr ptr)` method, the export that we will generate look like:

```csharp
[UnmanagedCallersOnly]
public static int QueryInterface(IntPtr* self, Guid* __arg1, IntPtr* __arg2)
{
    var handleAddress = *(self + 1);
    var handle = GCHandle.FromIntPtr(handleAddress);
    var obj = (IUnknown)handle.Target;

    var result = obj.QueryInterface(*__arg1, out var __local2);

    *__arg2 = __local2;

    return result;
}
```

Because the methods are instance methods, we add the `IntPtr* self` argument. Also, if the function in the managed interface is decorated with the `in`/`out`/`ref` keyword, we declare the argument as a pointer type because `UnmanagedCallersOnly` methods do not support `in`/`out`/`ref`.

The resulting code to generate the exports is:

```csharp
var parameterList = new StringBuilder();

parameterList.Append("IntPtr* self");

foreach (var parameter in method.Parameters)
{
    var isPointer = parameter.RefKind == RefKind.None ? "" : "*";
    parameterList.Append($", {parameter.Type}{isPointer} __arg{parameter.Ordinal}");
}

exports.AppendLine($"            [UnmanagedCallersOnly]");
exports.AppendLine($"            public static {method.ReturnType} {method.Name}({parameterList})");
exports.AppendLine($"            {{");
exports.AppendLine($"                var handle = GCHandle.FromIntPtr(*(self + 1));");
exports.AppendLine($"                var obj = ({interfaceName})handle.Target;");
exports.Append($"                ");

if (!method.ReturnsVoid)
{
    exports.Append("var result = ");
}

exports.Append($"obj.{method.Name}(");

for (int i = 0; i < method.Parameters.Length; i++)
{
    if (i > 0)
    {
        exports.Append(", ");
    }

    if (method.Parameters[i].RefKind == RefKind.In)
    {
        exports.Append($"*__arg{i}");
    }
    else if (method.Parameters[i].RefKind is RefKind.Out)
    {
        exports.Append($"out var __local{i}");
    }
    else
    {
        exports.Append($"__arg{i}");
    }
}

exports.AppendLine(");");

for (int i = 0; i < method.Parameters.Length; i++)
{
    if (method.Parameters[i].RefKind is RefKind.Out)
    {
        exports.AppendLine($"                *__arg{i} = __local{i};");
    }
}

if (!method.ReturnsVoid)
{
    exports.AppendLine($"                return result;");
}

exports.AppendLine($"            }}");

exports.AppendLine();
exports.AppendLine();
```

For the function pointers, given the same method as before, we want to generate:

```csharp
*(vtable + 1) = (IntPtr)(delegate* unmanaged<IntPtr*, Guid*, IntPtr*>)&Exports.QueryInterface;
```

The code to generate it is:

```csharp
var sourceArgsList = new StringBuilder();
sourceArgsList.Append("IntPtr _");

for (int i = 0; i < method.Parameters.Length; i++)
{
    sourceArgsList.Append($", {method.Parameters[i].OriginalDefinition} a{i}");
}

functionPointers.Append($"            *(vtable + {delegateCount}) = (IntPtr)(delegate* unmanaged<IntPtr*");

for (int i = 0; i < method.Parameters.Length; i++)
{
    functionPointers.Append($", {method.Parameters[i].Type}");

    if (method.Parameters[i].RefKind != RefKind.None)
    {
        functionPointers.Append("*");
    }
}

if (method.ReturnsVoid)
{
    functionPointers.Append(", void");
}
else
{
    functionPointers.Append($", {method.ReturnType}");
}

functionPointers.AppendLine($">)&Exports.{method.Name};");

delegateCount++;
```

Once we've done that for every method in the interface, we just need to replace the values in our template and add the generated source file:

```csharp
sourceBuilder.Replace("{typeName}", typeName);
sourceBuilder.Replace("{visibility}", visibility);
sourceBuilder.Replace("{exports}", exports.ToString());
sourceBuilder.Replace("{interfaceName}", interfaceName);
sourceBuilder.Replace("{delegateCount}", delegateCount.ToString());
sourceBuilder.Replace("{functionPointers}", functionPointers.ToString());

context.AddSource($"{symbol.ContainingNamespace?.Name ?? "_"}.{symbol.Name}.g.cs", sourceBuilder.ToString());
```

And that's it, our source generator is now ready.

# Using the generated code

To use our source generator, we can declare the `IUnknown`, `IClassFactory`, and `ICorProfilerCallback` interfaces, and decorate them with the `[NativeObject]` attribute:

```csharp
[NativeObject]
public interface IUnknown
{
    HResult QueryInterface(in Guid guid, out IntPtr ptr);
    int AddRef();
    int Release();
}
```

```csharp
[NativeObject]
internal interface IClassFactory : IUnknown
{
    HResult CreateInstance(IntPtr outer, in Guid guid, out IntPtr instance);
    HResult LockServer(bool @lock);
}
```

```csharp
[NativeObject]
public unsafe interface ICorProfilerCallback : IUnknown
{
    HResult Initialize(IntPtr pICorProfilerInfoUnk);

    // 70+ methods, stripped for brevity
}
```

Then we implement `IClassFactory` and call `NativeObjects.IClassFactory.Wrap` to create the native wrapper and expose our instance of `ICorProfilerCallback`:

```csharp
public unsafe class ClassFactory : IClassFactory
{
    private NativeObjects.IClassFactory _classFactory;
    private CorProfilerCallback2 _corProfilerCallback;

    public ClassFactory()
    {
        _classFactory = NativeObjects.IClassFactory.Wrap(this);
    }

    // The native wrapper has an implicit cast operator to IntPtr
    public IntPtr Object => _classFactory;

    public HResult CreateInstance(IntPtr outer, in Guid guid, out IntPtr instance)
    {
        Console.WriteLine("[Profiler] ClassFactory - CreateInstance");

        _corProfilerCallback = new();
        
        instance = _corProfilerCallback.Object;
        return HResult.S_OK;
    }

    public HResult LockServer(bool @lock)
    {
        return default;
    }

    public HResult QueryInterface(in Guid guid, out IntPtr ptr)
    {
        Console.WriteLine("[Profiler] ClassFactory - QueryInterface - " + guid);

        if (guid == KnownGuids.ClassFactoryGuid)
        {
            ptr = Object;
            return HResult.S_OK;
        }

        ptr = IntPtr.Zero;
        return HResult.E_NOTIMPL;
    }

    public int AddRef()
    {
        return 1; // TODO: do actual reference counting
    }

    public int Release()
    {
        return 0; // TODO: do actual reference counting
    }
}
```

And expose it in `DllGetClassObject`:

```csharp
public class DllMain
{
    private static ClassFactory Instance;

    [UnmanagedCallersOnly(EntryPoint = "DllGetClassObject")]
    public static unsafe int DllGetClassObject(void* rclsid, void* riid, nint* ppv)
    {
        Console.WriteLine("[Profiler] DllGetClassObject");

        Instance = new ClassFactory();
        *ppv = Instance.Object;

        return 0;
    }
}
```

And finally, we can implement our instance of `ICorProfilerCallback`:

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

        // TODO: To be implemented in next article

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
    // Automatically generated by the IDE
}
```

If we run it with a test application, we can see that the functions are working as expected:

```
[Profiler] DllGetClassObject
[Profiler] ClassFactory - CreateInstance
[Profiler] ICorProfilerCallback2 - QueryInterface
[Profiler] ICorProfilerCallback2 - Initialize
Hello, World!
```

In the next step, we will take care of the last missing piece of the puzzle: implementing the `ICorProfilerCallback.Initialize` method, and retrieving the instance of `ICorProfilerInfo`. Then we will have everything we need to actually interact with the profiler API.
