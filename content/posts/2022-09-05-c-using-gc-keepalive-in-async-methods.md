---
url: c-using-gc-keepalive-in-async-methods-8d20fd79f0a0
canonical_url: https://medium.com/@kevingosse/c-using-gc-keepalive-in-async-methods-8d20fd79f0a0
title: 'Using GC.KeepAlive in async methods'
subtitle: GC.KeepAlive may not work the way you intend when using it in async methods.
date: 2022-09-05
description: ""
tags:
- dotnet
- async
author: Kevin Gosse
thumbnailImage: /images/c-using-gc-keepalive-in-async-methods-8d20fd79f0a0-1.webp
---

# [C#] Using GC.KeepAlive in async methods

Can you figure out what's wrong with this code?

```csharp
var taskCompletionSource = new TaskCompletionSource();

MyDelegateType myDelegate = () => taskCompletionSource.SetResult();

NativeMethods.MyNativeMethod(myDelegate);

await taskCompletionSource.Task;

GC.KeepAlive(myDelegate);
```

It turns out that when running in an actual application, there is a probability that `myDelegate` will get collected despite the call to [`GC.KeepAlive`](https://learn.microsoft.com/en-us/dotnet/api/system.gc.keepalive?view=net-6.0&WT.mc_id=DT-MVP-5003493).

This code was posted [in an issue in the dotnet/runtime repository](https://github.com/dotnet/runtime/issues/74966), and I followed it closely because I really couldn't tell what the error was. If you can't either, then stick around and let's figure out what's happening.

# GC.KeepAlive

First, let's remind what is `GC.KeepAlive` and why it's needed here.

As you probably already know, the GC is responsible for freeing memory when objects are not used anymore. It does so by tracking references: if there are no reachable references to a given object, then it can be cleaned away.

The GC is smart enough to know the exact point in a method where an object stops being referenced. For instance:

```csharp
void Test()
{
  var obj = new Object();
  DoStuffWithObj(obj);
  // At this point, the GC knows the object isn't used anymore and may collect obj
  DoStuff();
}
```

As soon as `DoStuffWithObj` is called, and assuming that method doesn't store a reference to `obj` somewhere, the object may be garbage collected.

> Note: If you want to test this behavior, make sure to run your code in Release mode, without a debugger. In Debug mode, the lifetime of the instances is extended until the end of the scope, to make debugging easier.

Normally, this is not something developers have to worry about. But things get trickier when you start calling native methods. Managed methods are decorated with special metadata that allows the GC to track the lifetime of the local variables. There is no such thing in native methods, so by default the GC considers that the native code does not hold any reference to managed code (doing the opposite could lead to memory leaks). From an object lifetime standpoint, calling a native method is effectively the same as calling a method with a weak reference:

```csharp
static void Test()
{
    var obj = new object();
    MyNativeMethod(new WeakReference(obj));
}

static void MyNativeMethod(WeakReference obj)
{
    // The native method does not keep the object alive
}
```

So what happens if the garbage collector decides to run while the native method is executing?

```csharp
[MethodImpl(MethodImplOptions.AggressiveOptimization)]
static void Test()
{
    var obj = new object();

    // Run a garbage collection in 1 second
    Task.Delay(1000).ContinueWith(_ => { GC.Collect(); });

    // As soon as MyNativeMethod is called, there are no more references to obj
    MyNativeMethod(new WeakReference(obj));
}

static void MyNativeMethod(WeakReference obj)
{
    // The native method does not keep the object alive

    // Doing stuff
    Thread.Sleep(2000);

    // Trying to use the object
    Console.WriteLine("Is alive: " + obj.IsAlive); // Is alive: False
}
```

As we can see here, the GC might collect the object while the native method is running, which would probably cause an access violation if the native code then tries to use it.

Let's go on a little tangent and talk about why I needed to add `[MethodImpl(MethodImplOptions.AggressiveOptimization)]` to this example.

Since around .NET Core 3, the tiered JIT is enabled by default. Because of it, the first 30 invocations of a method use a less optimized version of the code. Incidentally, that less optimized version is not as aggressive at tracking the lifetime of objects, and the lifetime of `obj` ends up being extended until the end of the method, ruining the example. `[MethodImpl(MethodImplOptions.AggressiveOptimization)]` forces the tiered JIT to emit an optimized version of the method during the first invocation.

Here's what happens if I remove it and run the example multiple times:

{{<image classes="fancybox center" src="/images/c-using-gc-keepalive-in-async-methods-8d20fd79f0a0-1.webp" >}}

Neat uh? The tiered JIT has been around for a few years but my mind is still blown away by the fact that a method can suddenly change its behavior through the execution of a program.

So, when calling a native method and giving it a reference to a managed object, the GC might collect that object at any point in time. How do we prevent that? There are multiple ways, and one of them is `GC.KeepAlive`.

The most counter-intuitive fact about `GC.KeepAlive` is that you don't put it *before* the place where you need it but *after*:

```csharp
[MethodImpl(MethodImplOptions.AggressiveOptimization)]
static void Test()
{
    var obj = new object();

    // Run a garbage collection in 1 second
    Task.Delay(1000).ContinueWith(_ => { GC.Collect(); });

    // As soon as MyNativeMethod is called, there are no more references to obj
    MyNativeMethod(new WeakReference(obj));
    
    GC.KeepAlive(obj); // Keep obj alive until this point
}

static void MyNativeMethod(WeakReference obj)
{
    // The native method does not keep the object alive

    // Doing stuff
    Thread.Sleep(2000);

    // Trying to use the object
    Console.WriteLine("Is alive: " + obj.IsAlive); // Is alive: True
}
```

This is because `GC.KeepAlive` does not actually do anything. You can just think of it as a syntactic sugar to insert a reference to an object. We could achieve the exact same result with an empty method:

```csharp
[MethodImpl(MethodImplOptions.AggressiveOptimization)]
static void Test()
{
    var obj = new object();

    // Run a garbage collection in 1 second
    Task.Delay(1000).ContinueWith(_ => { GC.Collect(); });

    MyNativeMethod(new WeakReference(obj));

    MyGCKeepAlive(obj);
}

[MethodImpl(MethodImplOptions.NoInlining)]
static void MyGCKeepAlive(object obj)
{
    // Nothing
}
```

But we have to add the `[MethodImpl(MethodImplOptions.NoInlining)]` attribute to prevent the JIT from optimizing away the empty method, which would remove our fake reference. At least with `GC.KeepAlive` you don't have to worry about this kind of stuff.

# GC.KeepAlive in async methods

Now that we know what is `GC.KeepAlive` and how to use it, let's go back to the code that prompted this article:

```csharp
var taskCompletionSource = new TaskCompletionSource();

MyDelegateType myDelegate = () => taskCompletionSource.SetResult();

NativeMethods.MyNativeMethod(myDelegate);

await taskCompletionSource.Task;

GC.KeepAlive(myDelegate);
```

The author of this code is trying to track the completion of an asynchronous native method. To do so, they create a delegate that will complete a `TaskCompletionSource`, then give that delegate to the native code. Assumedly, the native code will call that delegate at the end of the asynchronous operation, thus completing the TCS.

Because the GC can't track the lifetime of `myDelegate` through the invocation of `NativeMethods.MyNativeMethod`, the author of this code added a call to `GC.KeepAlive(myDelegate)` at the end of the method. Yet, the application crashed with an access violation. What's going on?

Let's start by making sure we can reproduce the issue. Just like before, we're going to use a weak reference to simulate the behavior of the native call:

```csharp
[MethodImpl(MethodImplOptions.AggressiveOptimization)]
static async Task AsyncMethod()
{
    // Run a garbage collection in 1 second
    Task.Delay(1000).ContinueWith(_ => { GC.Collect(); });

    var taskCompletionSource = new TaskCompletionSource();

    Action myDelegate = () => taskCompletionSource.SetResult();

    MyNativeMethod(new WeakReference(myDelegate));

    await taskCompletionSource.Task;

    GC.KeepAlive(myDelegate);
}

static void MyNativeMethod(WeakReference myDelegate)
{
    // Simulate an asynchronous operation
    Task.Run(() =>
    {
        // Do some stuff
        Thread.Sleep(2000);

        Console.WriteLine("Is alive: " + myDelegate.IsAlive); // Is alive: False
    });
}
```

And indeed, this code displays `Is alive: False`, meaning that the delegate got collected despite the call to `GC.KeepAlive`.

As hinted by the title of the article, the issue is that we're using `GC.KeepAlive` inside of an async method. If we rewrite the code to wait synchronously instead…

```csharp
[MethodImpl(MethodImplOptions.AggressiveOptimization)]
static void SyncMethod()
{
    // Run a garbage collection in 1 second
    Task.Delay(1000).ContinueWith(_ => { GC.Collect(); });

    var taskCompletionSource = new TaskCompletionSource();

    Action myDelegate = () => taskCompletionSource.SetResult();

    MyNativeMethod(new WeakReference(myDelegate));

    taskCompletionSource.Task.Wait(); // Wait synchronously on the TCS

    GC.KeepAlive(myDelegate);
}
```

… then the code displays `Is alive: True` as expected.

To explain this, we must dig a bit into how async methods work. At compilation time they are rewritten into a state machine. Our previous example would look like this (simplified) :

```csharp
class StateMachine
{
    private int _state;
    private Action _myDelegate;

    public void MoveNext()
    {
        switch (_state)
        {
            case 0: //Initial state
            {
                Task.Delay(1000).ContinueWith(_ => { GC.Collect(); });

                var taskCompletionSource = new TaskCompletionSource();

                _myDelegate = () => taskCompletionSource.SetResult();

                MyNativeMethod(new WeakReference(_myDelegate));

                _state = 1; // Jump to the next step when the execution resume

                // Ask taskCompletionSource.Task to call MoveNext again when it completes
                taskCompletionSource.Task.GetAwaiter().OnCompleted(() => MoveNext());

                return;
            }

            case 1: // Continuation of the "await taskCompletionSource.Task"
            {
                GC.KeepAlive(_myDelegate);
                return;
            }
        }
    }
}
```

Suddenly, it becomes apparent that `GC.KeepAlive` is not going to work as intended. It is on a different branch than the call to `MyNativeMethod`, and it's just going to reference an object that the state machine is already referencing anyway. But then why isn't our state machine keeping `myDelegate` alive?

To understand that, let's walk the reference chain. `myDelegate` is referenced by the async state machine. The async state machine is referenced by the task continuation (implicitly created when calling `await taskCompletionSource.Task;`). The task continuation is referenced by the `TaskCompletionSource`. And the `TaskCompletionSource` is referenced by… nobody once the method exits.

Put in other words, when calling a synchronous method, the caller is responsible for continuing the workflow once the callee returns. When calling an asynchronous method, the responsibility is inverted: the caller returns, and the callee must call the continuation once the asynchronous operation is completed. But since the callee is a native method, it is not capable of keeping the managed reference alive.

The fix here is to allocate a GC handle to keep the reference alive:

```csharp
[MethodImpl(MethodImplOptions.AggressiveOptimization)]
static async Task AsyncMethod()
{
    // Run a garbage collection in 1 second
    Task.Delay(1000).ContinueWith(_ => { GC.Collect(); });

    var taskCompletionSource = new TaskCompletionSource();

    Action myDelegate = () => taskCompletionSource.SetResult();

    var handle = GCHandle.Alloc(myDelegate);

    MyNativeMethod(new WeakReference(myDelegate));

    await taskCompletionSource.Task;

    handle.Free();
}
```

You can think of the GC handle as a special reference that will always stay alive until `Free()` is called. The GC handle is going to keep the delegate alive, which references the `TaskCompletionSource`, which references the task continuation, which references the async state machine. And so the object graph is preserved until the end of the asynchronous operation.

# Is this a bug?

Honestly, I'm conflicted. The behavior makes perfect sense once you look at what's happening under the hood. But even with a good knowledge of how `async` works in C#, I'm fairly sure that I would have made the same mistake if I had to write this code. In fact, it's only after Stephen Toub commented in the github issue that I realized why the code was failing. On the other hand, adding special support for `GC.KeepAlive` in async methods would require a significant amount of work for something that is rarely used, so it's probably not worth the trouble. Maybe adding a warning when `GC.KeepAlive` is used in an async method could be a good middle ground.
