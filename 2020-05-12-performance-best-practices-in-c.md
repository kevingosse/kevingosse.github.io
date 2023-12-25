---
url: https://medium.com/@kevingosse/performance-best-practices-in-c-b85a47bdd93a
canonical_url: https://medium.com/@kevingosse/performance-best-practices-in-c-b85a47bdd93a
title: Performance best practices in C#
subtitle: Non-exhaustive list of code patterns to avoid in C#, either because they
  are risky or perform poorly.
slug: performance-best-practices-in-c
description: ""
tags:
- csharp
- performance
- best-practices
- software-development
- dotnet
author: Kevin Gosse
username: kevingosse
---

# Performance best practices in C#

As I recently had to compile a list of best practices in C# for Criteo, I figured it would be nice to share it publicly. The goal of this article is to provide a non-exhaustive list of code patterns to avoid, either because they’re risky or because they perform poorly. The list may seem a bit random because it’s out of context, but **all** the items have been spotted in our code at some point and have caused production issues. Hopefully this will serve as a warning and prevent you from making the same mistakes.

Also note that Criteo web services rely on high-performance code, hence the need to avoid inefficient code patterns. Some of those patterns wouldn’t make a noticeable difference in most applications.

Last but not least, some points have already been discussed in length in many articles (such as `ConfigureAwait`), so I do not elaborate on them. The goal is to have a list of points to keep in mind, rather than an in-depth technical description of each of them.

# Waiting synchronously on asynchronous code

**Don’t ever wait synchronously for non-completed tasks.** Including but not limited to: `Task.Wait`, `Task.Result`, `Task.GetAwaiter().GetResult()`, `Task.WaitAny`, `Task.WaitAll`.
As a more general advice, any synchronous dependency between two threadpool threads is susceptible to cause threadpool starvation. The causes of the phenomenon are described [in this article](https://medium.com/criteo-labs/net-threadpool-starvation-and-how-queuing-makes-it-worse-512c8d570527).

# ConfigureAwait

**If your code may be called from a synchronization context, use `ConfigureAwait(false)`** on each of your await calls.**

Note however that `ConfigureAwait` **only ever has a meaning when using the `await`** keyword**.

For instance, this code doesn’t make any sense:

```
// Using ConfigureAwait doesn't magically make this call safe
var result = ProcessAsync().ConfigureAwait(false).GetAwaiter().GetResult();
```
> *[Program.cs view raw](https://gist.githubusercontent.com/kevingosse/2d329b05c418c614c11f6505fb313f05/raw/9f32d2cfd7023ac6c40d52da9e9e67e53c977656/Program.cs)*

# async void

**Never use `async void`**. An exception thrown in an `async void` method is propagated to the synchronization context and usually ends up crashing the whole application.

If you can’t return a task in your method (for instance because you’re implementing an interface), move the async code to another method and call it:

```
interface IInterface
{
    void DoSomething();
}

class Implementation : IInterface
{
    public void DoSomething()
    {
        // This method can't return a Task,
        // delegate the async code to another method
        _ = DoSomethingAsync();
    }

    private async Task DoSomethingAsync()
    {
        await Task.Delay(100);
    }
}
```
> *[Program.cs view raw](https://gist.githubusercontent.com/kevingosse/20936a96e3b5177a0f224a470870a04e/raw/511c6e0c26e6d9a86ee78cb5597f7b2af19f4fc7/Program.cs)*

# Avoid async when possible

Out of habit/muscle memory, you might write:

```
public async Task CallAsync()
{
    var client = new Client();
    return await client.GetAsync();
}
```
> *[Program.cs view raw](https://gist.githubusercontent.com/kevingosse/9b2b39c1d0fc29c11a688cb14c470382/raw/5922e16b3b4039e5b853c2128aaeba0c444fe116/Program.cs)*

While the code is semantically correct, using the `async` keyword is not needed here and can have a significant overhead in hot paths. Try removing it whenever possible:

```
public Task CallAsync()
{
    var client = new Client();
    return client.GetAsync();
}
```
> *[Program.cs view raw](https://gist.githubusercontent.com/kevingosse/cc9870ad72be280b824a45388f5b62c7/raw/0c0b2ae5ecd2ddfeac7a263573c42f14972d831e/Program.cs)*

However, keep in mind that you can’t use that optimization when your code is wrapped in blocks (like `try/catch` or `using`) :

```
public async Task Correct()
{
    using (var client = new Client())
    {
        return await client.GetAsync();
    }
}

public Task Incorrect()
{
    using (var client = new Client())
    {
        return client.GetAsync();
    }
}
```
> *[Program.cs view raw](https://gist.githubusercontent.com/kevingosse/e06ecd897293b38eeaebb0a71f67fcaf/raw/9ba89b48970103fe9cde96e4bf95e29ae95d48d5/Program.cs)*

In the incorrect version, since the task isn’t awaited inside of the using block, the client might be disposed before the `GetAsync` call completes.

# Culture-sensitive comparisons

Unless you have a reason to use culture-sensitive comparisons, **always use ordinal comparisons** instead. While it doesn’t make much of a difference with en-US culture because of internal optimizations, the comparisons get one order of magnitude slower with other cultures (up to 2 orders of magnitude on Linux!). As string comparisons are a frequent operation in most applications, it quickly adds up.

# ConcurrentBag<T>

**Never use `ConcurrentBag<T>` without benchmarking**. This collection has been designed for very specific use-cases (when most of the time an item is dequeued by the thread that enqueued it) and suffers from important performance issues if used otherwise. If in need of a concurrent collection, **prefer `ConcurrentQueue<T>`**.

# ReaderWriterLock<T> / ReaderWriterLockSlim<T>

**Never use `ReaderWriterLock<T>`/`ReaderWriterLockSlim<T>` without benchmarking.** While it may be tempting to use this kind of specialized synchronization primitive when dealing with readers and writers, its cost is much higher than a simple `Monitor` (usable with the `lock` keyword). Unless the number of readers executing the critical section at the same time is *very* high, the concurrency won’t be enough to amortize the increased overhead, and the code will perform worse.

# Prefer Lambda functions instead of MethodGroup

Consider the following code:

```
public IEnumerable<int> GetItems()
{
    return _list.Where(i => Filter(i));
}
private static bool Filter(int element)
{
    return i % 2 == 0;
}
```
> *[Program.cs view raw](https://gist.githubusercontent.com/kevingosse/cdd073bf9803885c1ba7b7a130c257fc/raw/3f6081953a3fd436579cba903bb43e1c5e47986b/Program.cs)*

Resharper suggests to rewrite the code without a lambda function, which may look cleaner:

```
public IEnumerable<int> GetItems()
{
    return _list.Where(Filter);
}
private static bool Filter(int element)
{
    return i % 2 == 0;
}
```
> *[Program.cs view raw](https://gist.githubusercontent.com/kevingosse/bb798f0573c1726716e8935a2c4f76ca/raw/bfd015231f699e5e388370c8b3b478183e9e50a0/Program.cs)*

Unfortunately, doing so introduces a heap allocation at each call. Indeed, the call is compiled as:

```
public IEnumerable<int> GetItems()
{
    return _list.Where(new Predicate<int>(Filter));
}
private static bool Filter(int element)
{
    return i % 2 == 0;
}
```
> *[Program.cs view raw](https://gist.githubusercontent.com/kevingosse/6e143699784519f90def90fe498b80c1/raw/450c54fecd06d3b51dffba7a5c5312437de30e8e/Program.cs)*

This can have a significant impact if the code is called in a hot spot.

Using lambda functions triggers a compiler optimization that caches the delegate into a static field, removing that allocation. This only works if `Filter`** is static. If not, you may want to cache the delegate yourself:

```
private Predicate<int> _filter;

public Constructor()
{
    _filter = new Predicate<int>(Filter);
}

public IEnumerable<int> GetItems()
{
    return _list.Where(_filter);
}

private bool Filter(int element)
{
    return i % 2 == 0;
}
```
> *[Program.cs view raw](https://gist.githubusercontent.com/kevingosse/4c79a85461d495b4137e542301170df3/raw/b06daa365c0685a75809841231f584a3744cd7af/Program.cs)*

# Converting enums to string

Calling `Enum.ToString` in .net is very costly, as reflection is used internally for the conversion and calling a virtual method on struct causes boxing. As much as possible, this should be avoided.

Oftentimes, enums can actually be replaced by const strings:

```
// In both cases, you can use Numbers.One, Numbers.Two, ...
public enum Numbers
{
    One,
    Two,
    Three
}

public static class Numbers
{
    public const string One = "One";
    public const string Two = "Two";
    public const string Three = "Three";
}
```
> *[Program.cs view raw](https://gist.githubusercontent.com/kevingosse/0929083ca9df82a8125761eeee54b915/raw/a41fbe48b154307e64a9d0797e06f17e1c24edee/Program.cs)*

If you really need an enum, then consider caching the converted value in a dictionary to amortize the cost.

# Enum comparisons

**Note: this is not true anymore in .net core since version 2.1, the optimization is performed automatically by the JIT**

When using enums as flags, it may be tempting to use the `Enum.HasFlag` method:

```
[Flags]
public enum Options
{
    Option1 = 1,
    Option2 = 2,
    Option3 = 4
}

private Options _option;

public bool IsOption2Enabled()
{
    return _option.HasFlag(Options.Option2);
}
```
> *[Program.cs view raw](https://gist.githubusercontent.com/kevingosse/c5f84cb6324f322ad4993f950b5bdd83/raw/18bd1cf6d1eb8f740989e0dcade26b95d5c2c5f7/Program.cs)*

This code causes two boxing allocations: one to convert `Options.Option2` to `Enum`, and another one for the `HasFlag` virtual call on a struct. This makes this code disproportionately expensive. Instead, you should sacrifice readability and use binary operators:

```
public bool IsOption2Enabled()
{
    return (_option & Options.Option2) == Options.Option2;
}
```
> *[Program.cs view raw](https://gist.githubusercontent.com/kevingosse/5c063359a4c8a84739caaa1aae20b57b/raw/a80cd5fcdfc691688c9d63cff9b13d2de0d7de93/Program.cs)*

# Implement equality members for structs

When using struct in comparisons (for instance, when used as a key for a dictionary), you need to override the `Equals`/`GetHashCode` methods. The default implementation uses reflection and is very slow. The implementation generated by Resharper is usually good enough.

More information: [https://devblogs.microsoft.com/premier-developer/performance-implications-of-default-struct-equality-in-c/](https://devblogs.microsoft.com/premier-developer/performance-implications-of-default-struct-equality-in-c/?WT.mc_id=DT-MVP-5003493)

# Avoid unnecessary boxing when using structs with interfaces

Consider the following code:

```
public class IntValue : IValue
{
}

public void DoStuff()
{
    var value = new IntValue();

    LogValue(value);
    SendValue(value);
}

public void SendValue(IValue value)
{
    // ...
}

public void LogValue(IValue value)
{
    // ...
}
```
> *[Program.cs view raw](https://gist.githubusercontent.com/kevingosse/ebe6fcbcc188c8e4269faf552011864b/raw/06283933a0a4700ac38fb68d467db83af065cabc/Program.cs)*

It’s tempting to make `IntValue` a struct to avoid heap allocations. But since `AddValue` and `SendValue` expect an interface, and interfaces have reference semantics, the value will be boxed at every call, nullifying the benefits of the “optimization”. In fact, it will allocate even more than if `IntValue` was a class, since the value will be boxed independently for each call.

If you write an API and expect some values to be structs, try using generic methods:

```
public struct IntValue : IValue
{
}

public void DoStuff()
{
    var value = new IntValue();

    LogValue(value);
    SendValue(value);
}

public void SendValue<T>(T value) where T : IValue
{
    // ...
}

public void LogValue<T>(T value) where T : IValue
{
    // ...
}
```
> *[Program.cs view raw](https://gist.githubusercontent.com/kevingosse/6a2a4ef531116c2d919afe91a113dfa4/raw/c931d7c9a0bd22faafa711e88513c0d981c55021/Program.cs)*

While making those methods generic looks useless at first glance, it actually allows to avoid the boxing allocation when `IntValue` is a struct.

# CancellationToken subscriptions are always inlined

Whenever you cancel a `CancellationTokenSource`, all subscriptions will be executed inside of the current thread. This can lead to unplanned pauses or even subtle deadlocks.

```
var cts = new CancellationTokenSource();
cts.Token.Register(() => Thread.Sleep(5000));
cts.Cancel(); // This call will block during 5 seconds
```
> *[Program.cs view raw](https://gist.githubusercontent.com/kevingosse/d342dc59c38c9c63532999953239861b/raw/24e4f92d073536f813492fa1f8f8781c1874ddee/Program.cs)*

You cannot opt-out from this behavior. Therefore, when cancelling a `CancellationTokenSource`, ask yourself whether you can safely let your current thread be hijacked. If the answer is no, then wrap the call to `Cancel` inside of a `Task.Run` to execute it on the threadpool.

# TaskCompletionSource continuations are often inlined

Just like `CancellationToken` subscriptions, `TaskCompletionSource` continuations are often inlined. This is a nice optimization, but it can be the cause of subtle errors. For instance, consider the following program:

```
class Program
{
    private static ManualResetEventSlim _mutex = new ManualResetEventSlim();
    
    public static async Task Deadlock()
    {
        await ProcessAsync();
        _mutex.Wait();
    }
    
    private static Task ProcessAsync()
    {
        var tcs = new TaskCompletionSource<bool>();
        
        Task.Run(() =>
        {
            Thread.Sleep(2000); // Simulate some work
            tcs.SetResult(true);
            _mutex.Set();
        });
        
        return tcs.Task;
    }
    
    static void Main(string[] args)
    {
        Deadlock().Wait();
        Console.WriteLine("Will never get there");
    }
}
```
> *[Program.cs view raw](https://gist.githubusercontent.com/kevingosse/b7f2f53d55afb7b06ecfa69b28828a84/raw/887cf589dd2f3366dc78f5f33710e130bb9d4594/Program.cs)*

The call to `tcs.SetResult`** causes the continuation of `await ProcessAsync()`** to execute in the current thread. Therefore, the statement `_mutex.Wait()` is executed by the same thread that is supposed to call `_mutex.Set()`, resulting in a deadlock. This can be prevented by giving the `TaskCreationsOptions.RunContinuationsAsynchronously` parameter to the `TaskCompletionSource`.

**Unless you have a good reason to omit it, always use the `TaskCreationsOptions.RunContinuationsAsynchronously` parameter when creating a `TaskCompletionSource`.**

Beware: the code will also compile if you use `**TaskContinuationOptions**.RunContinuationsAsynchronously` instead of `**TaskCreationOptions**.RunContinuationsAsynchronously`, but the parameters will be ignored and the continuations will keep being inlined. This is a surprisingly common error, because `TaskContinuationOptions` comes before `TaskCreationOptions` in the auto-completion.

# Task.Run / Task.Factory.StartNew

Unless you have a reason to use `Task.Factory.StartNew`, always favor `Task.Run` to start a background task. `Task.Run` uses safer defaults, and more importantly it automatically unwraps the return task, which can prevent subtle errors with async methods. Consider the following program:

```
class Program
{
    public static async Task ProcessAsync()
    {
        await Task.Delay(2000);
        Console.WriteLine("Processing done");
    }
    
    static async Task Main(string[] args)
    {
        await Task.Factory.StartNew(ProcessAsync);
        Console.WriteLine("End of program");
        Console.ReadLine();
    }
}
```
> *[Program.cs view raw](https://gist.githubusercontent.com/kevingosse/c0cc1806eff7800e23ff9fec92db5fbb/raw/ee05162180efbc979b8614a4dbb68377b7bdea31/Program.cs)*

Despite the appearances, “End of program” will be displayed before “Processing done”. This is because `Task.Factory.StartNew` is going to return a `Task<Task>`, and the code only waits on the outer task. Correct code would be either `await Task.Factory.StartNew(ProcessAsync).Unwrap()`** or `await Task.Run(ProcessAsync)`.

There are only three legitimate use-cases for `Task.Factory.StartNew`:

* Starting a task on a different scheduler

* Executing the task on a dedicated thread (using `TaskCreationOptions.LongRunning`)

* Queuing the task on the threadpool global queue (using `TaskCreationOptions.PreferFairness`)


