---
url: the-hidden-trap-of-fixed-buffers-in-csharp
title: 'The hidden trap of fixed buffers in C#'
subtitle: After all, if they're marked as unsafe, there's gotta be a reason, right?
summary: After all, if they're marked as unsafe, there's gotta be a reason, right?
date: 2026-09-29
tags:
- dotnet
author: Kevin Gosse
thumbnailImage: /images/2026-09-29-the-hidden-trap-of-fixed-buffers-in-csharp-1.png
---

# An ordinary day of performance optimization...

While optimizing the licensing code used in ReSharper, I ran into an interesting problem. To give a bit of context, the licensing code was written before `BigInteger` was added to .NET Framework, so it uses an old open-source version that was originally written by Chew Keong Tan. The original isn't findable anymore, but there are still a few forks [here and there](https://github.com/bazzilic/BigInteger/blob/master/BigInteger/BigInteger.cs).

It does the job, but it caused some issues in ReSharper because it wasn't written with performance in mind. More specifically, the class looks roughly like:

```csharp
public class BigInteger
{
    private const int maxLength = 70;
    private uint[] data;
    private int dataLength;

    public BigInteger()
    {
        data = new uint[maxLength];
        dataLength = 1;
    }

    // ...
}
```

What's wrong with it? This is a class, so allocated on the heap, and it wraps a chonky `uint[]` array of size 70. For every mathematical operation (addition, multiplication, modulo, you name it), a new instance of `BigInteger` is created, and that new instance causes the allocation of a new array. The ReSharper licensing code uses cryptography, which means _a lot_ of such operations, so the allocations add up quickly to the point that it became visible in profiling traces.

To fix it, I simply decided to rewrite it as a struct, with an embedded array:

```csharp
public unsafe struct BigInteger
{
    private const int maxLength = 70;
    private fixed uint data[maxLength];
    private int dataLength;

    public BigInteger()
    {
        dataLength = 1;
    }

    // ...
}
```

If you're not familiar with fixed arrays, it's a feature that stores the array inline, directly inside the struct. 

![Heap array vs fixed buffer](/images/2026-09-29-fixed-buffer-layout.svg)

In modern C# you would use the `InlineArray` attribute instead, which is safer and more elegant:

```csharp
public struct BigInteger
{
    private const int maxLength = 70;
    private Data data;
    private int dataLength;

    public BigInteger()
    {
        dataLength = 1;
    }

    // ...

    [InlineArray(maxLength)]
    private struct Data
    {
        private uint _element;
    }
}
```

Unfortunately, ReSharper is limited to .NET Framework because it runs inside of Visual Studio, so we're stuck with fixed arrays.


Rewriting the code was easy enough in appearance. And yet, when testing, I realized that the whole feature was broken. Debugging was quite difficult for a simple reason: I had no idea what intermediary results the crypto code is meant to produce (and I still don't), so I had no way to tell when the results started diverging. Still, after much persistence, I narrowed it down to one surprising behavior of fixed arrays.

Rather than disclosing it right away, take a minute to consider the following code (or ask your agent to, if you stopped reading code):

```csharp
for (int i = 0; i < 5; i++)
{
    var a = new StructWithFixedBuffer();
    Console.Write($"{a.buffer[0]} ");
    a.buffer[0]++;
}

unsafe struct StructWithFixedBuffer
{
    public fixed long buffer[1];
}
```

If you read it carefully, you noticed that the increment is done on the discarded instance of the struct, and therefore has no effect. The output will be:

```
0 0 0 0 0
```

Now, let's make one tiny change to the code:

```csharp
for (int i = 0; i < 5; i++)
{
    var a = new StructWithFixedBuffer();
    Console.Write($"{a.buffer[0]} ");
    a.buffer[0]++;
}

unsafe struct StructWithFixedBuffer
{
    public fixed long buffer[1];

    public StructWithFixedBuffer()
    {
        // Nothing to see here
    }
}
```

If you haven't noticed, I've added an empty constructor to the struct. That's the only change between the two versions. Now run it again, and the output will be:

```
0 1 2 3 4
```

What's happening is that the buffer is no longer zeroed when creating a new instance: each iteration sees the value left over by the previous one. As you can imagine, I was as confused as you must be right now. How could adding a constructor to the struct (empty or not) have any effect on whether the fixed buffer is zeroed? I took it to Twitter and Egor Bogatov (one of the JIT engineers at Microsoft) didn't have an explanation either.

{{<image classes="fancybox center" src="/images/2026-09-29-the-hidden-trap-of-fixed-buffers-in-csharp-1.png" >}}

So, what's really happening under the hood? It's time to dig into the C# specification.

# The part where we dig into the C# specification (it's that part)

It turns out it's not actually a bug. When I first encountered this issue, one year ago, it was a different world. I opened the C# specification, tried looking for a few relevant keywords, found nothing that could explain this behavior, and concluded that it was a bug. Fast-forward one year, agents have taken over the world, and after I asked Claude to look into the JIT code to explain the bug to me, it demonstrated that the behavior was _by design_ and produced the receipts.

A few features in C# are making the problem more dangerous and harder to understand. So first, let's revert to C# 7.3 by adding those lines to the csproj:

```xml
<PropertyGroup>
    <LangVersion>7.3</LangVersion>
</PropertyGroup>
```

Now, the code doesn't compile anymore:

```
error CS8370: Feature 'parameterless struct constructors' is not available in C# 7.3. Please use language version 10.0 or greater.
```

We fix it by adding an argument to the constructor, and while we're at it we add a field to assign the value to:


```csharp
for (int i = 0; i < 5; i++)
{
    var a = new StructWithFixedBuffer(42);
    Console.Write($"{a.buffer[0]} ");
    a.buffer[0]++;
}

unsafe struct StructWithFixedBuffer
{
    public fixed long buffer[1];
    public int value;

    public StructWithFixedBuffer(int val)
    {
        value = val;
    }
}
```

When executed, the code still outputs `0 1 2 3 4`. Now, notice something. If we remove the assignment to the field in the constructor...

```csharp
unsafe struct StructWithFixedBuffer
{
    public fixed long buffer[1];
    public int value;

    public StructWithFixedBuffer(int val)
    {
        // Nothing
    }
}
```

... then we get a compilation error:

```
error CS0171: Field 'Program.StructWithFixedBuffer.value' must be fully assigned before control is returned to the caller. Consider updating to language version '11.0' to auto-default the field.
```

That's right, when writing a constructor for a struct, we are required to assign a value to all fields. But then, why not the buffer? Because of a small quirk in the C# specification. More specifically, [section §23.8.4 about Definite assignment checking](https://github.com/dotnet/csharpstandard/blob/standard-v7/standard/unsafe-code.md#2384-definite-assignment-checking):

> Fixed-size buffers are not subject to definite assignment-checking (§9.4), and fixed-size buffer members are ignored for purposes of definite-assignment checking of struct type variables.
> When the outermost containing struct variable of a fixed-size buffer member is a static variable, an instance variable of a class instance, or an array element, the elements of the fixed-size buffer are automatically initialized to their default values (§9.3). In all other cases, the initial content of a fixed-size buffer is undefined.

"Definite assignment checking" is the name of the feature checking that every field of the struct is assigned during initialization. What it says, in plain words, is: the compiler doesn't enforce that you assign a value to your fixed buffer. If the struct lives on the heap (static field, class field, array element), the buffer is zeroed. For a local, all bets are off.

It was already kind of dangerous in older versions of C# (breaking news, unsafe code is dangerous), but it's made worse by auto-default structs. Since C# 11, you are not required to zero all fields of a struct anymore, the compiler will take care of it for you. _Except for fixed buffers._ So the following code is legal:

```csharp
unsafe struct StructWithFixedBuffer
{
    public fixed long buffer[1];
    public int value1;
    public int value2;

    public StructWithFixedBuffer(int val)
    {
        value1 = val;
    }
}
```

The compiler will zero the unassigned `value2` for you but will leave `buffer` alone.

But what about the constructor-less version?

```csharp
unsafe struct StructWithFixedBuffer
{
    public fixed long buffer[1];
    public int value1;
    public int value2;
}
```

In this case, when constructing the struct, the compiler will emit an `initobj` instruction, which is pretty much the equivalent of writing:

```
var a = default(StructWithFixedBuffer);
```

There, the JIT will zero the fields of the struct for you, _including the fixed buffer_. Talk about confusing.


# The fix

Once you understand the problem, the fix is fairly straightforward: ~~ask Microsoft to rewrite the C# spec~~, ~~ask Microsoft to convert Visual Studio to .NET Core so ReSharper can use InlineArray~~ manually zero the struct in the constructor.

```csharp
public unsafe struct BigInteger
{
    private const int maxLength = 70;
    private fixed uint data[maxLength];
    private int dataLength;

    public BigInteger()
    {
        dataLength = 1;
        fixed (uint* d = data)
            new Span<uint>(d, maxLength).Clear(); // Use Span to get vectorization for free
    }

    // ...
}
```