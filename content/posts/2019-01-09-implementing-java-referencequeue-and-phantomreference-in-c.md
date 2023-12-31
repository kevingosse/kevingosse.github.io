---
url: implementing-java-referencequeue-and-phantomreference-in-c-827d7141b6e4
canonical_url: https://medium.com/criteo-engineering/implementing-java-referencequeue-and-phantomreference-in-c-827d7141b6e4
title: Implementing Java ReferenceQueue and PhantomReference in C#
subtitle: The Java ReferenceQueue is a mechanism that lets you know when an object has been GC'd. How hard can it be to reimplement it in .NET?
summary: The Java ReferenceQueue is a mechanism that lets you know when an object has been GC'd. How hard can it be to reimplement it in .NET?
date: 2019-01-09
tags:
- dotnet
- java
- garbage-collection
author: Kevin Gosse
---

After reading [Konrad Kokosa's article on Java PhantomReference](http://tooslowexception.com/do-we-need-jvms-phantomreference-in-net/), I got reminded of a coding challenge a coworker gave me a few months ago, about implementing a `ReferenceQueue` in C#. The Java `ReferenceQueue` is a mechanism that allows the developer to know when an object has been garbage collected. One of the usages described by Konrad is the ability to cleanup native resources without keeping an object (and its graph of references) in memory. How hard can it be to reimplement it in .NET?

We know a way to be notified when an object is collected: its finalizer will be called. Let's try to hack something together:

```csharp
public class PhantomReference
{
    private readonly ReferenceQueue _queue;

    protected PhantomReference(ReferenceQueue queue)
    {
        _queue = queue;
    }
   
    ~PhantomReference()
    {
        _queue.Notify(this);
    }
}

public class ReferenceQueue
{
    private readonly ConcurrentQueue<PhantomReference> _references;

    public ReferenceQueue()
    {
        _references = new ConcurrentQueue<PhantomReference>();
    }
   
    public PhantomReference Poll()
    {
        return _references.TryDequeue(out var value) ? value : null;
    }
    
    internal void Notify(PhantomReference value)
    {
        _references.Enqueue(value);
    }
}
```

We defined a base class, `PhantomReference`, and the `ReferenceQueue` class itself. The `PhantomReference` class defines a finalizer that adds the instance to the associated `ReferenceQueue`. Queued instances are kept until dequeued by calling `Poll`.

From there, we can implement the `PhantomReferenceFinalizer` class (see Konrad's article) on top of it:

```csharp
public class LargeObject
{
    public int Handle;
    
    // Plus some heavy stuff that we don't want to keep around once the object is no more referenced
}

public class PhantomReferenceFinalizer : PhantomReference<LargeObject>
{
    private int _handle;

    public PhantomReferenceFinalizer(ReferenceQueue<LargeObject> queue, LargeObject value)
        : base(queue)
    {
        _handle = value.Handle;
    }

    public void FinalizeResources()
    {
        Console.WriteLine("I'm cleaning handle " + _handle);
    }
}
```

The goal here is to have a way to clean the native resources held by `LargeObject` without keeping it in memory. Is it that simple? Well there's a critical flaw in our implementation, as illustrated by this code:

```csharp
class Program
{
    static void Main(string[] args)
    {
        var queue = new ReferenceQueue();

        InnerScope(queue);
    }
    
    static void InnerScope(ReferenceQueue queue)
    {
        var largeObject = new LargeObject { Handle = 42 };
        var phantomReference = new PhantomReferenceFinalizer(referenceQueue, largeObject);
        
        GC.Collect();
        GC.WaitForPendingFinalizers();

        Console.WriteLine(referenceQueue.Poll() == null); // Will display False even though largeObject is still alive!
        
        GC.KeepAlive(largeObject);
    }
}
```

What's the problem? We're actually tracking the lifecycle of our `PhantomReference` instead of our instance of `LargeObject`!

For this to work, we would need some kind of inverted `WeakReference`, where the object referenced (`LargeObject`) somehow keeps alive the object that holds the reference (`PhantomeReference`).

This unicorn actually exists in the .NET Framework. Its name is [`ConditionalWeakTable`](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.conditionalweaktable-2?view=netframework-4.7.2&WT.mc_id=DT-MVP-5003493). It works like a key/value dictionary, except that a value is automatically removed when its key is garbage collected. With that tool in hand, we can finish writing our `ReferenceQueue`:

```csharp
public class PhantomReference<T> where T : class
{
    private readonly ReferenceQueue<T> _queue;

    public PhantomReference(ReferenceQueue<T> queue, T value)
    {
        _queue = queue;
        queue.Track(value, this);
    }

    ~PhantomReference()
    {
        _queue.Notify(this);
    }
}

public class ReferenceQueue<T>
    where T : class
{
    private readonly ConditionalWeakTable<T, PhantomReference<T>> _table;
    private readonly ConcurrentQueue<PhantomReference<T>> _references;

    public ReferenceQueue()
    {
        _references = new ConcurrentQueue<PhantomReference<T>>();
        _table = new ConditionalWeakTable<T, PhantomReference<T>>();
    }
        
    public PhantomReference<T> Poll()
    {
        return _references.TryDequeue(out var value) ? value : null;
    }
        
    internal void Notify(PhantomReference<T> reference)
    {
        _references.Enqueue(reference);
    }

    internal void Track(T value, PhantomReference<T> reference)
    {
        _table.Add(value, reference);
    }
}
```

Then we can put everything together and verify that it works properly:

```csharp
public class PhantomObjectFinalizer : PhantomReference<LargeObject>
{
    private int _handle;

    public PhantomObjectFinalizer(ReferenceQueue<LargeObject> queue, LargeObject value)
        : base(queue, value)
    {
        _handle = value.Handle;
    }

    public void FinalizeResources()
    {
        Console.WriteLine("I'm cleaning handle " + _handle);
    }
}

class Program
{
    static void Main(string[] args)
    {
        var referenceQueue = new ReferenceQueue<LargeObject>();

        InnerScope(referenceQueue);

        GC.Collect();
        GC.WaitForPendingFinalizers();

        PhantomReference<LargeObject> referenceFromQueue;

        while ((referenceFromQueue = referenceQueue.Poll()) != null) 
        {
            // Will display "I'm cleaning handle 42"
            ((PhantomObjectFinalizer)referenceFromQueue).FinalizeResources(); 
        }

        Console.WriteLine("Done");
        Console.ReadLine();
    }

    static void InnerScope(ReferenceQueue<LargeObject> referenceQueue)
    {
        var largeObject = new LargeObject { Handle = 42 };

        var phantomReference = new PhantomObjectFinalizer(referenceQueue, largeObject);

        GC.Collect();
        GC.WaitForPendingFinalizers();

        Console.WriteLine(referenceQueue.Poll() == null); // True

        GC.KeepAlive(largeObject);
    }
}

```

Whether this `ReferenceQueue` has any practical application is up to debate, but this was a pretty good excuse to talk about the `ConditionalWeakTable`. Who knows when you may need it?

But is the `ConditionalWeakTable` really mandatory? Actually, there is another way to implement the `PhantomReference`. If we think about it, we need two things:

* A way to know whether our target object is still alive

* A way to be notified when a garbage collection occurs, to know when to check

For the first condition, we can use a `WeakReference` to track the live status of an object. For the second, we already know of a GC notification system: the finalizer! All we need to do is checking whether the target object is still alive when the finalizer is called. If it is, then we re-register ourselves, to be notified at the next collection:

```csharp
public class PhantomReference<T> where T : class
{
    private readonly ReferenceQueue<T> _queue;

    private WeakReference _weakReference;

    public PhantomReference(ReferenceQueue<T> queue, T value)
    {
        _queue = queue;
        _weakReference = new WeakReference(value);
    }

    ~PhantomReference()
    {
        // A GC occured
        if (_weakReference.Target != null)
        {
            // Target is still alive, register for next collection
            GC.ReRegisterForFinalize(this);
            return;
        }

        // Target is dead, notify the queue
        _queue.Notify(this);
    }
}

public class ReferenceQueue<T>
    where T : class
{
    private readonly ConcurrentQueue<PhantomReference<T>> _references;

    public ReferenceQueue()
    {
        _references = new ConcurrentQueue<PhantomReference<T>>();
    }

    public void Notify(PhantomReference<T> reference)
    {
        _references.Enqueue(reference);
    }

    public PhantomReference<T> Poll()
    {
        return _references.TryDequeue(out var value) ? value : null;
    }
}
```

Of course, this is much less elegant than the `ConditionalWeakTable`. Still, this is a fun trick to play with.


