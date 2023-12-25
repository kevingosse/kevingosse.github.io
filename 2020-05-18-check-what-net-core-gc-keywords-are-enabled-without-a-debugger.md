---
url: https://medium.com/@kevingosse/check-what-net-core-gc-keywords-are-enabled-without-a-debugger-d616745c0d0e
canonical_url: https://medium.com/@kevingosse/check-what-net-core-gc-keywords-are-enabled-without-a-debugger-d616745c0d0e
title: Check what .net core GC keywords are enabled without a debugger
subtitle: How to inspect arbitrary values in the memory of a .net process on Linux..
  And find an unexpected bug in the process
slug: check-what-net-core-gc-keywords-are-enabled-without-a-debugger
description: ""
tags:
- dotnet
- dotnet-core
- software-development
- garbage-collection
- debugging
author: Kevin Gosse
username: kevingosse
---

# Check what .net core GC keywords are enabled without a debugger

***… and find an unexpected bug in the process***

[We recently run into performance issues](https://twitter.com/KooKiz/status/1238046441672761345), and many elements seemed to point towards [the GC verbose events](https://github.com/dotnet/diagnostics/blob/master/documentation/dotnet-trace-instructions.md) being enabled. To test this, I needed a way to check on live servers whether the events were activated.

**Note:** the whole article is about .net core on Linux. While the first part (implementation) can be transposed to any OS, I’m not sure the second part could be done on Windows without installing additional tools.

# Checking the implementation

The first step is to understand how the runtime decides whether those events are enabled or not. A cursory search [in the GC source code](https://raw.githubusercontent.com/dotnet/runtime/master/src/coreclr/src/gc/gc.cpp) points to the `EVENT_ENABLED` macro:

```
#if defined(FEATURE_EVENT_TRACE)
            // We are explicitly checking whether the event is enabled here.
            // Unfortunately some of the ETW macros do not check whether the ETW feature is enabled.
            // The ones that do are much less efficient.
            if (EVENT_ENABLED(GCAllocationTick_V3))
            {
                fire_etw_allocation_event (etw_allocation_running_amount[etw_allocation_index], gen_number, acontext->alloc_ptr);
            }

#endif //FEATURE_EVENT_TRACE
```
> *[gc.cpp view raw](https://gist.githubusercontent.com/kevingosse/85fa56a4cca449401e61c5017e2e320c/raw/29d079df1204deddfbadf9c12f53d4fd91790583/gc.cpp)*

The macro expands to `GCEventEnabled##name`:

```
#define EVENT_ENABLED(name) GCEventEnabled##name()
```
> *[gceventstatus.h view raw](https://gist.githubusercontent.com/kevingosse/b9641c5a164b93a25ff047f155811618/raw/76b64a47e70d22232b703571319860b2861cae74/gceventstatus.h)*

Which in turn is implemented as:

```
inline bool GCEventEnabled##name() { return GCEventStatus::IsEnabled(provider, keyword, level); }
```
> *[gceventstatus.h view raw](https://gist.githubusercontent.com/kevingosse/c1aebf96a6c68ee64b8cdea591fe3dd5/raw/d065745ef0b7aa659be73170b9ddd690c7d9c9ac/gceventstatus.h)*

So it looks like the filtering is happening in `GCEventStatus`. The `IsEnabled` method simply checks the value of `enabledLevels` and `enabledKeywords`:

```
__forceinline static bool IsEnabled(GCEventProvider provider, GCEventKeyword keyword, GCEventLevel level)
{
    assert(level >= GCEventLevel_None && level < GCEventLevel_Max);

    size_t index = static_cast<size_t>(provider);
    return (enabledLevels[index].LoadWithoutBarrier() >= level)
      && (enabledKeywords[index].LoadWithoutBarrier() & keyword);
}
```
> *[gceventstatus.h view raw](https://gist.githubusercontent.com/kevingosse/b4ccef5196b91dd107d93073b9cbc5cc/raw/26e2c45d1da473d33822b02fc67b90b3ca14af8e/gceventstatus.h)*

Both are declared as arrays with hard-coded size:

```
Volatile<GCEventLevel> GCEventStatus::enabledLevels[2] = {GCEventLevel_None, GCEventLevel_None};
Volatile<GCEventKeyword> GCEventStatus::enabledKeywords[2] = {GCEventKeyword_None, GCEventKeyword_None};
```
> *[gceventstatus.cpp view raw](https://gist.githubusercontent.com/kevingosse/ceed26e0b719ef7fd0f601f889c1132b/raw/2ca6b763a8f87bb25c7cce16c15daccc1116aad7/gceventstatus.cpp)*

If we can read the value of `enabledLevels` in a live process, we’ll know whether the GC events are enabled or not.

# Inspecting a live process

How to read a variable in a live process? Usually I’d use a debugger, but we were having trouble with LLDB on our servers and I was feeling playful, so I decided to try something else.

Given sufficient rights, it’s possible to read the memory of a process through the file `/proc/<pid>/mem`. Therefore all we have to do is to figure out the memory address of the `enabledLevels` array.

For that, we’re going to inspect the symbols, just like a real debugger would. On Linux, there are no PDBs, the symbols are embedded directly in the libraries. They can be extracted [by using the `nm` utility](https://linux.die.net/man/1/nm).

If you’re not building your own CLR, it’s likely that the symbols are not included. You can download them using the `dotnet symbol` command:

```
> dotnet symbol --modules -d -o ~/symbols /usr/share/dotnet/shared/Microsoft.NETCore.App/3.1.1/*
```

Then, we can use `nm` to locate the symbol for `GCEventStatus::enabledLevels`:

```
> nm -C ~/symbols/libcoreclr.so | grep GCEventStatus::enabledLevels
0000000000764ff0 b GCEventStatus::enabledLevels
```

Now we know that the `GCEventStatus::enabledLevels` array is mapped to the memory offset 0x764ff0. The offset is relative to the base address of the `libcoreclr.so` module. We can know what address the module is mapped to by reading the `/proc/<pid>/maps` file:

```
> cat /proc/2563/maps | grep libcoreclr.so
7fbf1df40000-7fbf1e18e000 r-xp 00000000 103:02 9699852                   /usr/share/dotnet/shared/Microsoft.NETCore.App/3.1.1/libcoreclr.so
7fbf1e18e000-7fbf1e18f000 rwxp 0024e000 103:02 9699852                   /usr/share/dotnet/shared/Microsoft.NETCore.App/3.1.1/libcoreclr.so
7fbf1e18f000-7fbf1e47e000 r-xp 0024f000 103:02 9699852                   /usr/share/dotnet/shared/Microsoft.NETCore.App/3.1.1/libcoreclr.so
7fbf1e47e000-7fbf1e47f000 r--p 0053e000 103:02 9699852                   /usr/share/dotnet/shared/Microsoft.NETCore.App/3.1.1/libcoreclr.so
7fbf1e47f000-7fbf1e639000 r-xp 0053f000 103:02 9699852                   /usr/share/dotnet/shared/Microsoft.NETCore.App/3.1.1/libcoreclr.so
7fbf1e639000-7fbf1e63a000 ---p 006f9000 103:02 9699852                   /usr/share/dotnet/shared/Microsoft.NETCore.App/3.1.1/libcoreclr.so
7fbf1e63a000-7fbf1e688000 r--p 006f9000 103:02 9699852                   /usr/share/dotnet/shared/Microsoft.NETCore.App/3.1.1/libcoreclr.so
7fbf1e688000-7fbf1e692000 rw-p 00747000 103:02 9699852                   /usr/share/dotnet/shared/Microsoft.NETCore.App/3.1.1/libcoreclr.so
```

The file is mapped at different locations, the base address should be the first one (0x7fbf1df40000).

Putting everything together, the `GCEventStatus::enabledLevels` array should be mapped at the address: 0x7fbf1df40000 (base address of libcoreclr.so) + 0x764ff0 (offset in the symbols) = 0x7fbf1e6a4ff0

For some reason, I couldn’t use the `hexdump` utility to read the `/proc/<pid>/mem` file. I’m no Linux expert but apparently this is expected as the file is special. Instead, I used `[dd](http://man7.org/linux/man-pages/man1/dd.1.html)`.

```
> dd bs=1 skip="$((0x7fbf1e6a4ff0))" count=8 status=none if="/proc/2563/mem" | od -An -vtu4
          5          5
```

* `bs=1`: sets the size of the blocks to 1 byte

* `skip="$((0x7fbf1e6a4ff0))"` : skips to address 0x7fbf1e6a4ff0

* `count=8`: reads 8 blocks (= 8 bytes = 2 integers)

* `status=none`: hides status messages

* `if="/proc/2563/mem”` : sets the input file

* `od -An -vtu4` : formats the output as 4 bytes unsigned integers

For convenience, I’ve wrote all the steps in a bash script:

```
#!/bin/bash
offset="0x"$(nm -C $2/libcoreclr.so | grep GCEventStatus::enabledLevels | cut -d ' ' -f1)
module="0x"$(cat /proc/$1/maps | grep -m 1 libcoreclr.so | cut -d '-' -f1)
dd bs=1 skip="$(($module + $offset))" count=8 status=none if="/proc/$1/mem" | od -An -vtu4
```
> *[checkGCKeywords.sh view raw](https://gist.githubusercontent.com/kevingosse/0d2768370fd8f0f88cd762be290d76f1/raw/67311e467635d10d2a39ed76cddccb28e2e34666/checkGCKeywords.sh)*

# The plot twist

Now we can test it with a simple console application. First we start the application and launch the script to check the value of the keywords:

```
> sudo ./gckeywords.sh 6487 /home/kgosse/symbols
          0         0
```

As expected, the value is 0, meaning that the keywords aren’t activated.

Then, we run `dotnet-trace collect --profile gc-collect -p 6487` and check the keywords again:

```
> sudo ./gckeywords.sh 6487 /home/kgosse/symbols
          4        0
```

The level is now 4 for one of the providers, which is “informational”. That’s the expected level for the `gc-collect` profile. Now let’s detach dotnet-trace and check again:

```
> sudo ./gckeywords.sh 6487 /home/kgosse/symbols
          5       5
```

That… was completely unexpected. Detaching dotnet-trace somehow activated the GC verbose events! If we attach dotnet-trace again:

```
> sudo ./gckeywords.sh 6487 /home/kgosse/symbols
          4      5
```

We see that the first provider switched back to informational. If we detach dotnet-trace again:

```
> sudo ./gckeywords.sh 6487 /home/kgosse/symbols
          5     5
```

… back to verbose. This gives us a strong lead to understand why the verbose events are mysteriously active on our servers.


