---
url: an-unexpected-profiler-trap
title: 'Too good to be true: an unexpected profiler trap'
subtitle: Sometimes the most convincing performance gains deserve a second look. A reminder that profiling data can be more subtle than it first appears.
summary: Sometimes the most convincing performance gains deserve a second look. A reminder that profiling data can be more subtle than it first appears.
date: 2026-02-20
tags:
- dotnet
- performance
- profiling
author: Kevin Gosse
thumbnailImage: /images/2026-02-20-an-unexpected-profiler-trap-1.png
---

I ran into an interesting gotcha with PerfView (but that applies to profiling in general).

My goal was, and still is, to optimize the startup of ReSharper in Visual Studio. This is a complicated process because it involves a lot of different small tasks, there isn't one nice and big bottleneck that takes 90% of the time and that you can focus your efforts on. Rather, it involves saving 1% here, 2% there, and letting all the small optimizations add up.

My target this time was the `ResolveContextManager.GetResolveResult` method, which appeared as a hotspot in my test scenario. After digging further in with Perfview, I saw that the cumulated CPU time was ~23 seconds (2.6% of the total overhead during startup), divided like this :

```
ResolveContextManager.GetResolveResult (23,191 ms)
    └ TargetFrameworkScope.GetResolveResult (13,230 ms)
    └ ResolveContextManager.TryFindAssemblyByLocation (1,248 ms)
    ... (cut for brevity)
```

More than half of the time was spent in `TargetFrameworkScope.GetResolveResult`, so it seemed like an obvious first target, and I started to focus on that particular function. Even though `ResolveContextManager.GetResolveResult` was the main caller, `TargetFrameworkScope.GetResolveResult` was also called from other places, for a total of 16,218 ms:

```
Callers of TargetFrameworkScope.GetResolveResult (16,218 ms)
    - ResolveContextManager.GetResolveResult (13,230 ms)
    - others (2,988 ms)
```

After close analysis, I concluded that some of the workload was duplicate, and so caching could be a quick win. I started with the most straightforward implementation, moving all the logic to the callback of a `ConcurrentDictionary`:

```csharp
public class TargetFrameworkScope
{
    private ConcurrentDictionary<AssemblyNameInfo, IAssemblyLocation> _cache = new();

    public IAssemblyLocation GetResolveResult(AssemblyNameInfo assemblyNameInfo)
    {
        return _cache.GetOrAdd(assemblyNameInfo, key => GetResolveResultNoCache(key));
    }
    
    private IAssemblyLocation GetResolveResultNoCache(AssemblyNameInfo assemblyNameInfo)
    {
        // Old logic
        ...
    }
}
```

Based on my previous analysis, I expected a 30-40% improvement, which wouldn't be enough for my goals but would be a good base for further optimizations. I was way, _way_ off.

```
TargetFrameworkScope.GetResolveResult (2,496 ms)
```

From 16,218 ms to 2,496 ms, that's an 85% improvement! Very surprising, I'm always skeptical when an optimization is better than expected. If it's too good to be true, then it's usually not true. Still, in this case, I couldn't see anything wrong.

In any case, my original goal was `ResolveContextManager.GetResolveResult`, so I took a step back and checked `ResolveContextManager.GetResolveResult` again to see what else I could optimize.

```
ResolveContextManager.GetResolveResult (19,096 ms)
```

Hang on. `ResolveContextManager.GetResolveResult` was previously ~23 seconds. I cut the execution time of `TargetFrameworkScope.GetResolveResult` by 85%, or almost 11 seconds. How come the execution time of `ResolveContextManager.GetResolveResult` was only reduced by 4 seconds?

Expanding the calltree revealed the answer:

```
ResolveContextManager.GetResolveResult (19,096 ms)
    └ System.Collections.Concurrent.ConcurrentDictionary`2.GetOrAdd (8,400 ms)
    └ ResolveContextManager.TryFindAssemblyByLocation (1,862 ms)
    ... (cut for brevity)
```

By making `TargetFrameworkScope.GetResolveResult` a one-liner, I made it possible for the JIT to inline it, thus hiding it completely from PerfView. When looking at the callers, `ResolveContextManager.GetResolveResult` was gone and only the other path remained:

```
Callers of TargetFrameworkScope.GetResolveResult (2,496 ms)
    - others (2,496 ms)
```

My optimization actually reduced the execution time from 16,218 ms to 10,896 ms (the 8,400 ms from the inlined `ConcurrentDictionary` plus the 2,496 ms from the other caller). A mere 32% improvement, on the lower end of what I was expecting to get.

So what's the moral here? Honestly I don't know. I only noticed what was happening because my goal was a bigger function/process. It makes me look back at other times when I focused on a particular hotspot and optimized it. Did I actually optimize it, or did I run into the same trap without noticing? Hard to know.

Of course, there are often a few ways to avoid ending up in this situation. When possible, use benchmarks, though they also have their own pitfalls so they should be used in addition to a more macro approach, rather than replacing it entirely. If possible, make sure to measure high level metrics, such as the total execution time of a particular feature in your application. But there again, it only makes sense if what you're optimizing is making up for a significant portion of the total time. In my case, the function was responsible for ~2% of the CPU usage during the startup of Visual Studio. This is a very chaotic phase, where background tasks are started in random order and where I/O costs vary greatly from one run to the other. 2% is well within the noise margin, so it is not possible to measure reliably that way, except by doing tens or hundreds of runs which would take hours.

I guess from now on, I'll make a habit of marking the target function as non-inlinable whenever using a profiler to measure the effectiveness of my optimizations.