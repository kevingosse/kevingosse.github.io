---
url: writing-a-net-profiler-in-c-part-2-8039da001e43
canonical_url: https://medium.com/@kevingosse/writing-a-net-profiler-in-c-part-2-8039da001e43
title: Writing a .NET profiler in C#  —  Part 2
subtitle: Using NativeAOT to write a .NET profiler in C#, learning many things about
  native interop in the process.
summary: Part 2 of the series about using NativeAOT to write a .NET profiler in C#, learning many things about native interop in the process. In this part, we improve the code from the previous article by using instance methods instead of static methods.
date: 2023-01-11
description: ""
tags:
- dotnet
- csharp
- nativeaot
- profiler
author: Kevin Gosse
---

[In the first part](/writing-a-net-profiler-in-c-part-1-d3978aae9b12), we saw how to mimic the layout of a COM object, and use it to expose a fake instance of IClassFactory. It worked nicely, but our solution used static methods, so it wouldn't be convenient to track the state of the objects whenever multiple instances are expected. It would be great if we could map our COM object to an actual instance of an object in .NET.

At this point, our code looks like:

```csharp
public class DllMain
{
    private static ClassFactory Instance;

    [UnmanagedCallersOnly(EntryPoint = "DllGetClassObject")]
    public static unsafe int DllGetClassObject(void* rclsid, void* riid, nint* ppv)
    {
        Console.WriteLine("Hello from the profiling API");

        // Allocate the chunk of memory for the vtable pointer + the pointers to the 5 methods
        var chunk = (IntPtr*)NativeMemory.Alloc(1 + 5, (nuint)IntPtr.Size);

        // Pointer to the vtable
        *chunk = (IntPtr)(chunk + 1);

        // Pointers to each method of the interface
        *(chunk + 1) = (IntPtr)(delegate* unmanaged<IntPtr, Guid*, IntPtr*, int>)&QueryInterface;
        *(chunk + 2) = (IntPtr)(delegate* unmanaged<IntPtr, int>)&AddRef;
        *(chunk + 3) = (IntPtr)(delegate* unmanaged<IntPtr, int>)&Release;
        *(chunk + 4) = (IntPtr)(delegate* unmanaged<IntPtr, IntPtr, Guid*, IntPtr*, int>)&CreateInstance;
        *(chunk + 5) = (IntPtr)(delegate* unmanaged<IntPtr, bool, int>)&LockServer;

        *ppv = (IntPtr)chunk;

        return HResult.S_OK;
    }

    [UnmanagedCallersOnly]
    public static unsafe int QueryInterface(IntPtr self, Guid* guid, IntPtr* ptr)
    {
        Console.WriteLine("QueryInterface");
        *ptr = IntPtr.Zero;
        return 0;
    }

    [UnmanagedCallersOnly]
    public static int AddRef(IntPtr self)
    {
        Console.WriteLine("AddRef");
        return 1;
    }

    [UnmanagedCallersOnly]
    public static int Release(IntPtr self)
    {
        Console.WriteLine("Release");
        return 1;
    }

    [UnmanagedCallersOnly]
    public static unsafe int CreateInstance(IntPtr self, IntPtr outer, Guid* guid, IntPtr* instance)
    {
        Console.WriteLine("CreateInstance");
        *instance = IntPtr.Zero;
        return 0;
    }

    [UnmanagedCallersOnly]
    public static int LockServer(IntPtr self, bool @lock)
    {
        return 0;
    }
}
```

Ideally, what we would like is an actual object with instance methods, like this:

```csharp
public class ClassFactory
{
    public unsafe int QueryInterface(IntPtr self, Guid* guid, IntPtr* ptr)
    {
        Console.WriteLine("QueryInterface");
        *ptr = IntPtr.Zero;
        return 0;
    }

    public int AddRef(IntPtr self)
    {
        Console.WriteLine("AddRef");
        return 1;
    }

    public int Release(IntPtr self)
    {
        Console.WriteLine("Release");
        return 1;
    }

    public unsafe int CreateInstance(IntPtr self, IntPtr outer, Guid* guid, IntPtr* instance)
    {
        Console.WriteLine("CreateInstance");
        *instance = IntPtr.Zero;
        return 0;
    }

    public int LockServer(IntPtr self, bool @lock)
    {
        return 0;
    }
}
```

However, the native side can only call methods decorated with the `UnmanagedCallersOnly` attribute, and this attribute can only be applied on static methods. So we will need a set of static methods, and a way to retrieve an instance of an object from those static methods.

The key to achieve this is the `self` argument of those methods. Because we are mimicking the layout of a C++ object, the address of the instance of the native object is passed as first argument. We can use it to retrieve our managed object and call the non-static version of the method. For instance:

```csharp
public unsafe class ClassFactory
{
    private static Dictionary<IntPtr, ClassFactory> _instances = new();

    public ClassFactory()
    {
        // Allocate the chunk of memory for the vtable pointer + the pointers to the 5 methods
        var chunk = (IntPtr*)NativeMemory.Alloc(1 + 5, (nuint)IntPtr.Size);

        // Pointer to the vtable
        *chunk = (IntPtr)(chunk + 1);

        // Pointers to each method of the interface
        *(chunk + 1) = (IntPtr)(delegate* unmanaged<IntPtr, Guid*, IntPtr*, int>)&QueryInterfaceNative;

        // [...] (stripped for brevity

        _instances.Add((IntPtr)chunk, this);
    }

    public int QueryInterface(Guid* guid, IntPtr* ptr)
    {
        Console.WriteLine("QueryInterface");
        *ptr = IntPtr.Zero;
        return 0;
    }

    // [...] (same for other instance methods of ClassFactory)

    [UnmanagedCallersOnly]
    public static int QueryInterfaceNative(IntPtr self, Guid* guid, IntPtr* ptr)
    {
        var instance = _instances[self];

        return instance.QueryInterface(guid, ptr);
    }

    // [...] (same for other static methods of ClassFactory)
}
```

In the constructor, we add the instance of `ClassFactory` to a static dictionary, with the address of the associated native object. In the static `QueryInterfaceNative` method, we retrieve that instance from the static dictionary, and call the non-static `QueryInterface` method.

It works, but it's a shame to do a dictionary lookup every time a method is called. Plus, we need to handle concurrency (probably by using a `ConcurrentDictionary`). Is there a better solution?

We already have a pointer to a native object, so it would be great if that native object could store a pointer to the managed object. Something like this:

```csharp
public ClassFactory()
{
    // Allocate the chunk of memory for the vtable pointer + the address of the managed object + the pointers to the 5 methods
    var chunk = (IntPtr*)NativeMemory.Alloc(2 + 5, (nuint)IntPtr.Size);

    // Pointer to the vtable
    *chunk = (IntPtr)(chunk + 2);

    // Pointer to the managed object
    *(chunk + 1) = &this;

    // [...]
}
```

If we had that, then from the static method it would just be a matter of fetching the pointer to the managed object:

```csharp
[UnmanagedCallersOnly]
public static unsafe int QueryInterfaceNative(IntPtr* self, Guid* guid, IntPtr* ptr)
{
    var instance = *(ClassFactory*)(self + 1);

    return instance.QueryInterface(guid, ptr);
}
```

But `&this` won't compile*, for good reasons: a managed object can be moved at any time by the garbage collector, so the pointer could become invalid at the next garbage collection.

> *: I lied. If you use the latest version of C#, then you can take the address of `this`:
> 
> ```csharp
> var classFactory = this;
> *(chunk + 1) = (nint)(nint*)&classFactory;
> ```
>  
> But this is unsafe for the aforementioned reasons, so please don't unless you know what you're doing.

You may be tempted to pin the object to solve this problem, but you can't pin an object that has references to another managed object, so that's not good either.

What we need is a kind of pinned reference to a managed object, and fortunately `GCHandle` provides exactly that. If we allocate a `GCHandle` pointing to a managed object, we can use `GCHandle.ToIntPtr` to get a fixed address associated to that handle, and `GCHandle.FromIntPtr` to retrieve the handle from that address. Therefore, what we can do is:

```csharp
public ClassFactory()
{
    // Allocate the chunk of memory for the vtable pointer + the address of the managed object + the pointers to the 5 methods
    var chunk = (IntPtr*)NativeMemory.Alloc(2 + 5, (nuint)IntPtr.Size);

    // Pointer to the vtable
    *chunk = (IntPtr)(chunk + 2);

    // Pointer to the managed object
    var handle = GCHandle.Alloc(this);
    *(chunk + 1) = GCHandle.ToIntPtr(handle);

    // [...]
}
```

Then we can retrieve the handle and the associated object from the static method:

```csharp
[UnmanagedCallersOnly]
public static unsafe int QueryInterfaceNative(IntPtr* self, Guid* guid, IntPtr* ptr)
{
    var handleAddress = *(self + 1);
    var handle = GCHandle.FromIntPtr(handleAddress);
    var instance = (ClassFactory)handle.Target;

    return instance.QueryInterface(guid, ptr);
}
```

Wrapping everything together, our `ClassFactory` now looks like:

```csharp
public unsafe class ClassFactory
{
    public ClassFactory()
    {
        // Allocate the chunk of memory for the vtable pointer + the address of the managed object + the pointers to the 5 methods
        var chunk = (IntPtr*)NativeMemory.Alloc(2 + 5, (nuint)IntPtr.Size);

        // Pointer to the vtable
        *chunk = (IntPtr)(chunk + 2);

        // Pointer to the managed object
        var handle = GCHandle.Alloc(this);
        *(chunk + 1) = GCHandle.ToIntPtr(handle);

        *(chunk + 2) = (IntPtr)(delegate* unmanaged<IntPtr*, Guid*, IntPtr*, int>)&Exports.QueryInterface;
        *(chunk + 3) = (IntPtr)(delegate* unmanaged<IntPtr*, int>)&Exports.AddRef;
        *(chunk + 4) = (IntPtr)(delegate* unmanaged<IntPtr*, int>)&Exports.Release;
        *(chunk + 5) = (IntPtr)(delegate* unmanaged<IntPtr*, IntPtr, Guid*, IntPtr*, int>)&Exports.CreateInstance;
        *(chunk + 6) = (IntPtr)(delegate* unmanaged<IntPtr*, bool, int>)&Exports.LockServer;

        Object = (IntPtr)chunk;
    }

    public IntPtr Object { get; }

    public int QueryInterface(Guid* guid, IntPtr* ptr)
    {
        Console.WriteLine("QueryInterface");
        *ptr = IntPtr.Zero;
        return 0;
    }

    public int AddRef()
    {
        Console.WriteLine("AddRef");
        return 1;
    }

    public int Release()
    {
        Console.WriteLine("Release");
        return 1;
    }

    public int CreateInstance(IntPtr outer, Guid* guid, IntPtr* instance)
    {
        Console.WriteLine("CreateInstance");
        *instance = IntPtr.Zero;
        return 0;
    }

    public int LockServer(bool @lock)
    {
        Console.WriteLine("LockServer");
        return 0;
    }

    private class Exports
    {
        [UnmanagedCallersOnly]
        public static int QueryInterface(IntPtr* self, Guid* guid, IntPtr* ptr)
        {
            var handleAddress = *(self + 1);
            var handle = GCHandle.FromIntPtr(handleAddress);
            var obj = (ClassFactory)handle.Target;

            return obj.QueryInterface(guid, ptr);
        }


        [UnmanagedCallersOnly]
        public static int AddRef(IntPtr* self)
        {
            var handleAddress = *(self + 1);
            var handle = GCHandle.FromIntPtr(handleAddress);
            var obj = (ClassFactory)handle.Target;

            return obj.AddRef();
        }

        [UnmanagedCallersOnly]
        public static int Release(IntPtr* self)
        {
            var handleAddress = *(self + 1);
            var handle = GCHandle.FromIntPtr(handleAddress);
            var obj = (ClassFactory)handle.Target;

            return obj.Release();
        }
        
        [UnmanagedCallersOnly]
        public static unsafe int CreateInstance(IntPtr* self, IntPtr outer, Guid* guid, IntPtr* instance)
        {
            var handleAddress = *(self + 1);
            var handle = GCHandle.FromIntPtr(handleAddress);
            var obj = (ClassFactory)handle.Target;

            return obj.CreateInstance(outer, guid, instance);
        }

        [UnmanagedCallersOnly]
        public static int LockServer(IntPtr* self, bool @lock)
        {
            var handleAddress = *(self + 1);
            var handle = GCHandle.FromIntPtr(handleAddress);
            var obj = (ClassFactory)handle.Target;

            return obj.LockServer(@lock);
        }
    }
}
```

(note that I moved the static methods to a nested class to avoid name collisions)

And we can use it from our entry point:

```csharp
public class DllMain
{
    private static ClassFactory Instance;

    [UnmanagedCallersOnly(EntryPoint = "DllGetClassObject")]
    public static unsafe int DllGetClassObject(void* rclsid, void* riid, nint* ppv)
    {
        Instance = new ClassFactory();

        Console.WriteLine("Hello from the profiling API");

        *ppv = Instance.Object;

        return HResult.S_OK;
    }
}
```

What is left is doing this for `ICorProfilerCallback` and its ~70 methods. We're not going to do this by hand, so in the next article we will write a source generator to automate the process.
