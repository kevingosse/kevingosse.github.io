---
url: switching-back-to-the-ui-thread-in-wpf-uwp-in-modern-c-5dc1cc8efa5e
canonical_url: https://medium.com/criteo-engineering/switching-back-to-the-ui-thread-in-wpf-uwp-in-modern-c-5dc1cc8efa5e
title: Switching back to the UI thread in WPF/UWP, in modern C#
subtitle: Leveraging the async machinery to transparently switch to the UI thread when needed
summary: Leveraging the async machinery to transparently switch to the UI thread when needed.
date: 2020-02-28
description: ""
tags:
- dotnet
- wpf
- uwp
- async
author: Kevin Gosse
---

It may seem weird that in 2020 I would write an article on this subject. But I found a new approach to this problem that I had some fun experimenting with.

So what is the problem, to begin with?

# A quick reminder

In WinForms/WPF/UWP, you can only update controls from the UI thread. If you have code running in a background thread that needs to update some controls, you need to somehow switch to the UI thread. This is done using `Control.Invoke` in WinForms and the dispatcher in WPF/UWP. The usage is pretty simple:

```csharp
Dispatcher.BeginInvoke(new Action(UpdateControls));
```

(Note: In this article, I'm going to focus on WPF, but everything can be translated to WinForms or UWP)

I've often ended up in cases where it's not obvious in which thread I was. To handle those cases, I would encapsulate the "switch to UI thread" logic inside of the method that needs to update controls. To know whether I was already running in the UI thread or not, I would use the `Dispatcher.CheckAccess()` method:

```csharp
private void UpdateControls()
{
    if (!Dispatcher.CheckAccess())
    {
        // We're not in the UI thread, ask the dispatcher to call this same method in the UI thread, then exit
        Dispatcher.BeginInvoke(new Action(UpdateControls));
        return;
    }

    // We're in the UI thread, update the controls
    TextTime.Text = DateTime.Now.ToLongTimeString();
}
```

I really like it because I feel like it provides better separation of concerns, and I can call any method without worrying whether that method should run on the UI thread or not.

I've mostly stopped developing desktop apps before tasks and async programming became mainstream. Nowadays, async/await provides another less-verbose solution thanks to custom awaiters. Today I would write something like:

```csharp
private async Task UpdateControls()
{
    await Dispatcher.SwitchToUi();

    TextTime.Text = DateTime.Now.ToLongTimeString();
}
```

To make that possible, I would declare `SwitchToUi` as an extension method on the dispatcher. 
How do we do that? When implementing an awaiter, you need to provide two methods and one property:

- `GetResult`: When called, you should synchronously wait for the operation to complete (if it's not completed already), and return the result (if any). Here, we'll leave the method empty because it does not make sense (we're switching contexts, we're not waiting on any operation)

- `IsCompleted`: This property indicates whether the async operation is completed. In our case, we consider it's completed if we're already running in the UI thread

- `OnCompleted`: This method will be called if `IsCompleted` returned false. It contains a callback that you need to call when the operation completes. For our purpose, we'll ask the dispatcher to invoke the callback on the UI thread.

Put together, our helper looks like:

```csharp
public static class DispatcherExtensions
{
    public static SwitchToUiAwaitable SwitchToUi(this Dispatcher dispatcher)
    {
        return new SwitchToUiAwaitable(dispatcher);
    }

    public struct SwitchToUiAwaitable : INotifyCompletion
    {
        private readonly Dispatcher _dispatcher;

        public SwitchToUiAwaitable(Dispatcher dispatcher)
        {
            _dispatcher = dispatcher;
        }

        public SwitchToUiAwaitable GetAwaiter()
        {
            return this;
        }

        public void GetResult()
        {
        }

        public bool IsCompleted => _dispatcher.CheckAccess();

        public void OnCompleted(Action continuation)
        {
            _dispatcher.BeginInvoke(continuation);
        }
    }
}
```

This is a trick that has been documented in various forms, for instance [on the blog of Thomas Levesque](https://thomaslevesque.com/2015/11/11/explicitly-switch-to-the-ui-thread-in-an-async-method/).

# OK I knew all of that already

Now things start to become interesting. A few days ago, I was playing with async method builders and custom task types [as part of a joke](https://twitter.com/KooKiz/status/1232599451279200262). A few hours later, I saw a question on StackOverflow about switching to the UI thread… And everything clicked together. What we used is `AsyncMethodBuilder` to fill the same use-case as before but in a less verbose way?

`AsyncMethodBuilder` is an API that allows you to returns custom task types in async methods. It's how `ValueTask` are implemented. The API gives you control over how calls are awaited inside of the method, but also when and on what thread the async method starts executing. That's what we're going to take advantage of.

How would that work? First, we need to declare a custom task type. We want to keep it simple, so we'll just make a wrapper around a `TaskCompletionSource`. We will also declare an implicit cast operator to `Task`, so that it can be used seamlessly with other APIs.
 We also add the `AsyncMethodBuilder` attribute, which tells the compiler what method builder to use when an async method returns this type of task. In our case, we'll name our method builder `UiTaskMethodBuilder`.

Note: If you're running on .net framework, you need to reference the [System.Threading.Tasks.Extensions](https://www.nuget.org/packages/System.Threading.Tasks.Extensions/) nuget package to use the `AsyncMethodBuilder` attribute.

```csharp
[AsyncMethodBuilder(typeof(UiTaskMethodBuilder))]
public class UiTask
{
    internal TaskCompletionSource<object> Promise { get; } = new TaskCompletionSource<object>();

    public Task AsTask() => Promise.Task;

    public TaskAwaiter<object> GetAwaiter()
    {
        return Promise.Task.GetAwaiter();
    }

    public static implicit operator Task(UiTask task) => task.AsTask();
}

```

For the method builder, there are a few methods we have to implement (it's all duck-typing, no interface to guide us):

* `Start`: As the name indicates, this is called at the beginning of the async method. We are given an instance of the async state machine and are expected to call `MoveNext` when we're ready to start the execution. In our implementation, we're going to check whether we are running on the UI thread. If not, we switch to it before calling `MoveNext`:

```csharp
public void Start<TStateMachine>(ref TStateMachine stateMachine)
    where TStateMachine : IAsyncStateMachine
{
    if (!_dispatcher.CheckAccess())
    {
        _dispatcher.BeginInvoke(new Action(stateMachine.MoveNext));
    }
    else
    {
        stateMachine.MoveNext();
    }
}
```

* `SetStateMachine`: This one is pretty obscure, I'm not even sure when it's supposed to be called. We're not going to need it so we'll leave it empty.

* `Task`: This property exposes the instance of our custom UI task that will be returned by the async method

* `SetResult` and `SetException`: Either of these methods is called at the end of the async method, to set the result. We can simply map them to the `TaskCompletionSource` in our UI task:

```csharp
public UiTask Task { get; } = new UiTask();

public void SetResult()
{
    Task.Promise.SetResult(null);
}

public void SetException(Exception exception)
{
    Task.Promise.SetException(exception);
}
```

* `AwaitOnCompleted` and `AwaitUnsafeOnCompleted`: Those methods are called whenever there's an `await` call inside of the async method. `AwaitUnsafeOnCompleted` will be called it the awaiter implements `ICriticalNotifyCompletion`, and `AwaitOnCompleted` will be called otherwise. The difference between the two is only about how the execution context is flown. In our case, we're not going to worry about it. We call `OnComplete`/`UnsafeOnCompleted` on the awaiter, to know when it's complete, and we make sure to resume execution on the UI thread:

```csharp
public void AwaitOnCompleted<TAwaiter, TStateMachine>(
    ref TAwaiter awaiter,
    ref TStateMachine stateMachine)
    where TAwaiter : INotifyCompletion
    where TStateMachine : IAsyncStateMachine
{
    awaiter.OnCompleted(ResumeAfterAwait(stateMachine));
}

public void AwaitUnsafeOnCompleted<TAwaiter, TStateMachine>(
    ref TAwaiter awaiter,
    ref TStateMachine stateMachine)
    where TAwaiter : ICriticalNotifyCompletion
    where TStateMachine : IAsyncStateMachine
{
    awaiter.UnsafeOnCompleted(ResumeAfterAwait(stateMachine));
}

private Action ResumeAfterAwait<TStateMachine>(TStateMachine stateMachine) where TStateMachine : IAsyncStateMachine
{
    return () =>
    {
        if (!_dispatcher.CheckAccess())
        {
            _dispatcher.BeginInvoke(new Action(stateMachine.MoveNext));
        }
        else
        {
            stateMachine.MoveNext();
        }
    };
}
```

Last but not least, we need to implement a static `Create` method, which will be called to instantiate our `UiTaskMethodBuilder` whenever an async method is invoked. Put together, our builder should look like:

```csharp
public class UiTaskMethodBuilder
{
    private readonly Dispatcher _dispatcher;

    public UiTaskMethodBuilder(Dispatcher dispatcher)
    {
        _dispatcher = dispatcher;
    }

    public void Start<TStateMachine>(ref TStateMachine stateMachine)
        where TStateMachine : IAsyncStateMachine
    {
        if (!_dispatcher.CheckAccess())
        {
            _dispatcher.BeginInvoke(new Action(stateMachine.MoveNext));
        }
        else
        {
            stateMachine.MoveNext();
        }
    }

    public static UiTaskMethodBuilder Create()
    {
        return new UiTaskMethodBuilder(Application.Current.Dispatcher);
    }

    public void SetStateMachine(IAsyncStateMachine stateMachine)
    {
    }

    public void SetResult()
    {
        Task.Promise.SetResult(null);
    }

    public void SetException(Exception exception)
    {
        Task.Promise.SetException(exception);
    }

    public UiTask Task { get; } = new UiTask();

    public void AwaitOnCompleted<TAwaiter, TStateMachine>(
        ref TAwaiter awaiter,
        ref TStateMachine stateMachine)
        where TAwaiter : INotifyCompletion
        where TStateMachine : IAsyncStateMachine
    {
        awaiter.OnCompleted(ResumeAfterAwait(stateMachine));
    }

    public void AwaitUnsafeOnCompleted<TAwaiter, TStateMachine>(
        ref TAwaiter awaiter,
        ref TStateMachine stateMachine)
        where TAwaiter : ICriticalNotifyCompletion
        where TStateMachine : IAsyncStateMachine
    {
        awaiter.UnsafeOnCompleted(ResumeAfterAwait(stateMachine));
    }

    private Action ResumeAfterAwait<TStateMachine>(TStateMachine stateMachine) where TStateMachine : IAsyncStateMachine
    {
        return () =>
        {
            if (!_dispatcher.CheckAccess())
            {
                _dispatcher.BeginInvoke(new Action(stateMachine.MoveNext));
            }
            else
            {
                stateMachine.MoveNext();
            }
        };
    }
}
```

Now we can go back to our `UpdateControls` method and remove the boilerplate code. Just by setting the return type to `UiTask`, we indicate our desire to run this method in the UI thread. If the method is invoked from a non-UI thread, the context will automatically change:

```csharp
private async UiTask UpdateControls()
{
    TextTime.Text = DateTime.Now.ToLongTimeString();
}
```

What do you think? Personally I really like how it indicates in the method signature that this call will run in the UI thread. If I was still implementing desktop applications, I would definitely use it. It adds some overhead and a few useless allocations, but for a non-server application it usually doesn't matter.

I believe it would be even better if we could do this with a custom attribute on `UpdateControls`, but it's not possible at the moment. It would however become possible [if this proposition was implemented](https://github.com/dotnet/csharplang/issues/1407).
