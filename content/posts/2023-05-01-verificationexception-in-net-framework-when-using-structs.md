---
url: verificationexception-in-net-framework-when-using-structs-6269eb3df448
canonical_url: https://medium.com/@kevingosse/verificationexception-in-net-framework-when-using-structs-6269eb3df448
title: VerificationException in .NET Framework when using structs
subtitle: A surprising error occuring when using C# 7.3 with partial trust.
summary: A surprising error occuring when using C# 7.3 with partial trust.
date: 2023-05-01
description: ""
tags:
- dotnet
- x64dbg
- cpp
author: Kevin Gosse
thumbnailImage: /images/verificationexception-in-net-framework-when-using-structs-6269eb3df448-2.webp
---

Consider the following code:

```csharp
private static readonly ReadonlyStruct Struct;

public static void Read()
{
    Console.WriteLine(Struct.Value);
}

public struct ReadonlyStruct
{
    public ReadonlyStruct(int value)
    {
        Value = value;
    }
        
    public int Value;
}
```

This looks pretty straightforward, right? We store a struct in a readonly field, and read it.

Yet, what happens if we try to run this program in .NET Framework under partial trust?

```csharp
static void Main(string[] args)
{
    var permissionSet = new PermissionSet(PermissionState.None);
    permissionSet.AddPermission(new SecurityPermission(SecurityPermissionFlag.Execution));
    permissionSet.AddPermission(new ReflectionPermission(PermissionState.Unrestricted));
            
    var appDomain = AppDomain.CreateDomain("ChildDomain", null, AppDomain.CurrentDomain.SetupInformation, permissionSet);

    try
    {
        appDomain.DoCallBack(RRead);
    }
    catch (VerificationException ex)
    {
        Console.WriteLine("Read failed: " + ex);
    }
}
```

The answer is… It crashes with a `VerificationException` and a scary message ("Operation could destabilize the runtime").

{{<image classes="fancybox center" src="/images/verificationexception-in-net-framework-when-using-structs-6269eb3df448-1.webp" >}}

The application stops crashing as soon as I remove the `readonly` keyword from the `Struct` field. Even weirder, it also stops crashing if you target an older version of C#, for instance by setting `<LangVersion>7</LangVersion>` in the csproj file.

Besides the obvious "why would anybody use partial trust nowadays" question, I was really surprised by this behavior. There's nothing obviously unsafe in this code, so I decided to dig further.

> To answer the part about using partial trust, we caught this error in the Datadog .NET tracer, which until now supported partial trust. The reason why we supported such an obsolete feature is simply that we never had a compelling reason to drop support. We now do.

# A brief reminder about partial trust

For the people who are lucky enough to never have to deal with partial trust, this is a feature that existed since the early days of .NET Framework. It allows to limit what type of code can be run in a given appdomain (or in the whole application), in theory to safely execute code coming from a potentially unsafe source (think for instance of a plugin system). Among other things, partial trust forbids the usage of the `unsafe` keyword, because otherwise it could be used to circumvent the limitations.

I didn't think much of this until I realized that forbidding `unsafe` is not as straightforward as it sounds. `unsafe` is a C# language feature, but partial trust operates at the IL level, where this concept does not exist. You could imagine that a special attribute is added when compiling unsafe C# code, but that wouldn't protect you from an assembly hand-crafted directly from IL. So how does the runtime detects the usage of `unsafe`? It does so by scanning the IL and checking for instructions that could *potentially* be unsafe (for instance, anything that directly loads an address), and restricts their usage to well-defined ("verifiable") conditions.

# So what is "potentially unsafe"?

Now that we know that code verification operates at the IL level, let's check what our sample code produces. I annotated it to make it easier to understand to people who are not familiar with IL:

```
.field private static initonly valuetype PartialTrust.ReadonlyStruct Struct

.method public hidebysig static void  Read() cil managed
{
  .maxstack  8
  // Load the address of the static Program.Struct field
  ldsflda    valuetype PartialTrust.ReadonlyStruct PartialTrust.Program::Struct
  // Read the Value field as an int32
  ldfld      int32 PartialTrust.ReadonlyStruct::Value
  // Invoke Console.WriteLine
  call       void [mscorlib]System.Console::WriteLine(int32)
  // Return
  ret
}
```

For comparison, here is the IL code emitted if I change the csproj to target C# 7:

```
.field private static initonly valuetype PartialTrust.ReadonlyStruct Struct

.method public hidebysig static void  Read() cil managed
{
  .maxstack  8
  // Load the value of the static Program.Struct field
  ldsfld    valuetype PartialTrust.ReadonlyStruct PartialTrust.Program::Struct
  // Read the Value field as an int32
  ldfld      int32 PartialTrust.ReadonlyStruct::Value
  // Invoke Console.WriteLine
  call       void [mscorlib]System.Console::WriteLine(int32)
  // Return
  ret
}
```

The only difference is the `ldsflda` instruction being replaced with `ldsfld`. The former loads the address of the field on the stack, while the latter loads the value of the field (and therefore makes a copy of the struct).

It seems like this change was introduced in C# 7.2. This version introduced readonly structs, which are guaranteed to be immutable and allow the compiler to perform additional optimizations (reading the struct by reference instead of having to do a defensive copy). In my sample, the struct is not actually readonly, but because it's stored in a readonly field it looks like it's enough for the compiler to perform the same optimization.

In short, until C# 7.1, the compiler makes a copy of the struct because it's stored in a readonly field and it must guarantee that no change is made to the original value. Starting with C# 7.2, the compiler is smarter about it and realizes that reading a field from the struct is not going to cause any side-effect, and therefore stops making a copy.

> Note: there are a number of other situations where C# 7.2+ stops making a defensive copy, if you decorate the struct with the readonly keyword. Likewise, they will cause a VerificationException in partial trust. But the sample I'm showing in this article is in my opinion more interesting because it does not use that new keyword, and the code compiles without any change with earlier versions of C#.

# Why is `ldsflda` an issue

So far we've explained two things:

* Partial trust verifies the IL to make sure the code isn't doing anything dangerous

* Starting from C# 7.2 the `ldsfld` instruction in this sample gets replaced with `ldsflda`, causing the struct to be read by reference instead of value

However, we still don't know why this is a problem at all. One critical thing to understand is that C# 7.2 is a compiler update. The runtime, and therefore the JIT compiler, hasn't been updated. The version of the JIT compiler bundled in .NET Framework does not expect the `ldsflda` instruction to be used in that situation.

I wanted to understand exactly what rule gets broken by the usage of `ldsflda`, so I checked the source code of the JIT. Partial trust has been removed in .NET Core, so we need to check the source code of .NET Framework. It's not publicly available, but you can get a close approximation [by checking the first commit of the CoreCLR repository on github](https://github.com/dotnet/coreclr/tree/ef1e2ab328087c61a6878c1e84f4fc5d710aebce), before it diverged too much.

By executing the code with a native debugger ([x64dbg](https://x64dbg.com)'s disassembly window is great, I recommend it), and cross-referencing the CoreCLR C++ code, I was eventually able to narrow it down to those few lines:

```c++
CORINFO_CLASS_HANDLE enclosingClass = pResolvedToken->hClass;
unsigned fieldFlags = fieldInfo.fieldFlags;
CORINFO_CLASS_HANDLE instanceClass = info.compClassHnd; // for statics, we imagine the instance is the current class.

bool isStaticField = ((fieldFlags & CORINFO_FLG_FIELD_STATIC) != 0);
if (mutator)  
{
    Verify(!(fieldFlags & CORINFO_FLG_FIELD_UNMANAGED), "mutating an RVA bases static");
    if ((fieldFlags & CORINFO_FLG_FIELD_FINAL))
    {
        Verify((info.compFlags & CORINFO_FLG_CONSTRUCTOR) &&
               enclosingClass == info.compClassHnd && info.compIsStatic == isStaticField,
               "bad use of initonly field (set or address taken)");
    }
}
```

{{<image classes="fancybox center" src="/images/verificationexception-in-net-framework-when-using-structs-6269eb3df448-2.webp" title="x64dbg is great to annotate the assembly" >}}

This code is invoked when the JIT compiler verifies a method and finds a `ldsflda` instruction. Specifically, our `VerificationException` is caused by this line:

```c++
Verify((info.compFlags & CORINFO_FLG_CONSTRUCTOR) &&
               enclosingClass == info.compClassHnd && info.compIsStatic == isStaticField,
               "bad use of initonly field (set or address taken)");
```

It verifies that `ldsflda` is used in the constructor of the class, as this is the only place where mutating a readonly field is allowed. Outside of that case, it assumes that the address of the field (returned by `ldsflda`) can be used to mutate the value, and so it forbids it. I'm not a compiler expert but I feel like flow analysis could prove that reading the address is safe in that case, but because this optimization didn't exist when that JIT was written, they had no reason to do so. When the optimization was introduced with C# 7.2, Microsoft apparently didn't feel like it was necessary to upgrade the JIT compiler, probably because partial trust was already considered as obsolete.

# The fix

If for some reason you still need to support partial trust, I recommend adding this flag to the csproj:

```xml
<Features>peverify-compat</Features>
```

It opts-out of those new incompatible optimizations. The name implies it was designed to maintain compatibility with the `peverify` tool, but it works with partial trust as well.
