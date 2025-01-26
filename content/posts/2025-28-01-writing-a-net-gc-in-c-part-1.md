---
url: 2025-28-01-writing-a-net-gc-in-c-part-1
title: Writing a .NET Garbage Collector in C#  -  Part 1
subtitle: Using NativeAOT to write a .NET GC in C#. An excuse to discover the internals of the .NET Garbage Collector.
summary: First part of a series of articles about writing a .NET Garbage Collector in C# using NativeAOT. This part sets the expectations and setups the project, dealing with the first difficulties.
date: 2025-01-28
tags:
- dotnet
- nativeaot
- gc
author: Kevin Gosse
thumbnailImage: /images/2025-28-01-writing-a-net-gc-in-c-part-1-3.png
---

If you read my articles, you probably know that I like playing with NativeAOT a lot, especially to use C# in places where it wasn't possible before. We already [wrote a simple profiler](https://minidump.net/writing-a-net-profiler-in-c-part-5/), this time we will go a step further and try to write a Garbage Collector in C#.

Of course, this won't result in anything usable in production. Building a performant and fully-featured GC would take hundreds of hours of work, and using a managed language for that is a poor choice (can you imagine your GC being randomly interrupted by its own internal garbage collection?). Still, this is a good excuse to learn more about the internals of the .NET Garbage Collector.

{{<image classes="fancybox center" src="/images/2025-28-01-writing-a-net-gc-in-c-part-1-1.png" >}}

Also, you may be wondering why NativeAOT is needed at all for this. Like for the profiler in .NET, there are two reasons why using "vanilla" .NET is impossible: first, the GC is initialized very early in the process, at a point in time when the CLR isn't ready to run managed code yet. Second, the GC would end up depending on itself, which is a bit of a chicken-and-egg problem. NativeAOT embeds its own CLR and does not interact with the system CLR, so it can be used to bypass these limitations.

# The standalone GC API

.NET added support for loading an external GC (so-called "standalone GC") in .NET Core 2.1. To use it, you need to build a DLL that exposes two methods:
 - `GC_Initialize`, called to initialize the GC
 - `GC_VersionInfo`, called to query the version of the API supported by the GC

The first step is to create a new .NET 9 library project, and set `<PublishAot>true</PublishAot>` in the csproj to enable NativeAOT. Then we can add a `DllMain` class (the name doesn't matter) and declare the two required methods:

```csharp
public class DllMain
{
    [UnmanagedCallersOnly]
    public static unsafe uint GC_Initialize(IntPtr clrToGC, IntPtr* gcHeap, IntPtr* gcHandleManager, GcDacVars* gcDacVars)
    {
        Console.WriteLine("GC_Initialize");
        return 0x80004005 /* E_FAIL */;
    }

    [UnmanagedCallersOnly]
    public static unsafe uint GC_VersionInfo(VersionInfo* versionInfo)
    {
        Console.WriteLine($"GC_VersionInfo {versionInfo->MajorVersion}.{versionInfo->MinorVersion}.{versionInfo->BuildVersion}");

        versionInfo->MajorVersion = 5;
        versionInfo->MinorVersion = 3;

        return 0;
    }

    [StructLayout(LayoutKind.Sequential)]
    public unsafe struct VersionInfo
    {
        public int MajorVersion;
        public int MinorVersion;
        public int BuildVersion;
        public byte* Name;
    }

    [StructLayout(LayoutKind.Sequential)]
    public readonly struct GcDacVars
    {
        public readonly byte Major_version_number;
        public readonly byte Minor_version_number;
        public readonly nint Generation_size;
        public readonly nint Total_generation_count;
    }
}
```

For now we don't do much, just writing a message to the console to confirm that the functions are properly called. In `GC_VersionInfo`, we also properly set the version of the API we support. Note that the `versionInfo` argument initially contains the version of the Execution Engine API provided by the CLR. This is useful if you want to write a GC that supports multiple versions of .NET.

The DLL can be published with NativeAOT by simply running:

```bash
dotnet publish -r win-x64
```

To load the custom GC into a .NET application, we need to copy it to the same folder as the application, and set the `DOTNET_GCName` environment variable to the name of the DLL. Alternatively, you can use `DOTNET_GCPath` that accepts a full path and therefore allows you to load the GC from another folder.

```bash
set DOTNET_GCName=ManagedDotnetGC.dll
```

Then we can run the application, and we are immediately greeted with our first crash:

{{<image classes="fancybox center" src="/images/2025-28-01-writing-a-net-gc-in-c-part-1-2.png" >}}

Sure, our "GC" is far from being functional, but we would have expected at least to see the messages we wrote to the console.

# Debugging the initialization

If we attach a debugger, we can see that the crash is an access violation in the `GC_Initialize` function, while trying to write to the console.

{{<image classes="fancybox center" src="/images/2025-28-01-writing-a-net-gc-in-c-part-1-3.png" >}}

However, if you look closely at the callstack, you might be able to spot something odd:

```
ManagedDotnetGC.dll!S_P_CoreLib_System_Threading_Volatile__Read_12<System___Canon>()
ManagedDotnetGC.dll!System_Console_System_Console__get_Out()
ManagedDotnetGC.dll!System_Console_System_Console__WriteLine_12()
ManagedDotnetGC.dll!ManagedDotnetGC_ManagedDotnetGC_DllMain__GC_Initialize()
[Inline Frame] ManagedDotnetGC.dll!GCHeapUtilities::InitializeDefaultGC()
ManagedDotnetGC.dll!InitializeDefaultGC()
ManagedDotnetGC.dll!InitializeGC()
[Inline Frame] ManagedDotnetGC.dll!InitDLL(void * hPalInstance)
ManagedDotnetGC.dll!RhInitialize(bool isDll)
ManagedDotnetGC.dll!InitializeRuntime()
[Inline Frame] ManagedDotnetGC.dll!Thread::EnsureRuntimeInitialized()
[Inline Frame] ManagedDotnetGC.dll!Thread::ReversePInvokeAttachOrTrapThread(ReversePInvokeFrame *)
ManagedDotnetGC.dll!RhpReversePInvokeAttachOrTrapThread2(ReversePInvokeFrame * pFrame)
ManagedDotnetGC.dll!ManagedDotnetGC_ManagedDotnetGC_DllMain__GC_VersionInfo()
coreclr.dll!`anonymous namespace'::LoadAndInitializeGC(const wchar_t * standaloneGCName, const wchar_t * standaloneGCPath)
coreclr.dll!InitializeGarbageCollector()
coreclr.dll!EEStartupHelper()
coreclr.dll!EEStartup()
coreclr.dll!EnsureEEStarted()
coreclr.dll!CorHost2::Start()
coreclr.dll!coreclr_initialize(const char * exePath, const char * appDomainFriendlyName, int propertyCount, const char * * propertyKeys, const char * * propertyValues, void * * hostHandle, unsigned int * domainId)
hostpolicy.dll!coreclr_t::create(const std::wstring & libcoreclr_path, const char * exe_path, const char * app_domain_friendly_name, const coreclr_property_bag_t & properties, std::unique_ptr<coreclr_t,std::default_delete<coreclr_t>> & inst)
hostpolicy.dll!`anonymous namespace'::create_coreclr()
hostpolicy.dll!corehost_main(const int argc, const wchar_t * * argv)
hostfxr.dll!execute_app(const std::wstring & impl_dll_dir, corehost_init_t * init, const int argc, const wchar_t * * argv)
hostfxr.dll!`anonymous namespace'::read_config_and_execute(const std::wstring & host_command, const host_startup_info_t & host_info, const std::wstring & app_candidate, const std::unordered_map<enum known_options,std::vector<std::wstring,std::allocator<std::wstring>>,known_options_hash,std::equal_to<enum known_options>,std::allocator<std::pair<enum known_options const ,std::vector<std::wstring,std::allocator<std::wstring>>>>> & opts, int new_argc, const wchar_t * * new_argv, host_mode_t mode, const bool is_sdk_command, wchar_t * out_buffer, int buffer_size, int * required_buffer_size)
hostfxr.dll!fx_muxer_t::handle_exec_host_command(const std::wstring & host_command, const host_startup_info_t & host_info, const std::wstring & app_candidate, const std::unordered_map<enum known_options,std::vector<std::wstring,std::allocator<std::wstring>>,known_options_hash,std::equal_to<enum known_options>,std::allocator<std::pair<enum known_options const ,std::vector<std::wstring,std::allocator<std::wstring>>>>> & opts, int argc, const wchar_t * * argv, int argoff, host_mode_t mode, const bool is_sdk_command, wchar_t * result_buffer, int buffer_size, int * required_buffer_size)
hostfxr.dll!fx_muxer_t::execute(const std::wstring host_command, const int argc, const wchar_t * * argv, const host_startup_info_t & host_info, wchar_t * result_buffer, int buffer_size, int * required_buffer_size)
hostfxr.dll!hostfxr_main_startupinfo(const int argc, const wchar_t * * argv, const wchar_t * host_path, const wchar_t * dotnet_root, const wchar_t * app_path)
TestApp.exe!exe_start(const int argc, const wchar_t * * argv)
TestApp.exe!wmain(const int argc, const wchar_t * * argv)
[Inline Frame] TestApp.exe!invoke_main()
TestApp.exe!__scrt_common_main_seh()
kernel32.dll!BaseThreadInitThunk()
ntdll.dll!RtlUserThreadStart()
```

Starting from the bottom, we can see a bunch of frames related to the initialization of the test application, then a call to `ManagedDotnetGC.dll!ManagedDotnetGC_ManagedDotnetGC_DllMain__GC_VersionInfo()`. So the runtime is calling `GC_VersionInfo`... but we're crashing in `GC_Initialize`? Looking more closely, we can see that `GC_VersionInfo` triggers the initialization of the NativeAOT runtime (`ManagedDotnetGC.dll!InitializeRuntime()`) which in turns initializes its own GC (`ManagedDotnetGC.dll!InitializeDefaultGC()`). But somehow, this ends up calling **our** `GC_Initialize` function, which is not supposed to be called at this point. Why would the initialization of the NativeAOT runtime trigger a call to our GC?

My initial theory was that the NativeAOT runtime was picking up our `DOTNET_GCName` environment variable and trying to use our custom GC instead of its own. After further research, while NativeAOT [has experimental support for standalone GC](https://github.com/dotnet/runtime/pull/91038), it is currently disabled by default (and needs to be enabled with a special flag). So, it can't be the cause of our crash.

After further investigation, the problem seems to have something to do with linking. In the NativeAOT runtime, [the `GC_Initialize` function is defined as an external symbol](https://github.com/dotnet/runtime/blob/main/src/coreclr/nativeaot/Runtime/gcheaputilities.cpp#L39-L46):

```cpp
// GC entrypoints for the linked-in GC. These symbols are invoked
// directly if we are not using a standalone GC.
extern "C" HRESULT LOCALGC_CALLCONV GC_Initialize(
    /* In  */ IGCToCLR* clrToGC,
    /* Out */ IGCHeap** gcHeap,
    /* Out */ IGCHandleManager** gcHandleManager,
    /* Out */ GcDacVars* gcDacVars
);
```

Because we also expose our own `GC_Initialize` function, the linker seems to be confused and links the wrong one. This can be verified by writing a simple console application that exposes a `GC_Initialize` method, and simply running it:

```csharp
internal class Program
{
    static void Main(string[] args)
    {
        Console.WriteLine("Hello, World!");
    }

    [UnmanagedCallersOnly(EntryPoint = "GC_Initialize")]
    public static int GC_Initialize()
    {
        return 0;
    }
}
```

If we publish it with NativeAOT then run it, it immediately crashes with the same access violation, even though we're never setting the `DOTNET_GCName` environment variable or even trying to invoke the `GC_Initialize` function.

Unfortunately I couldn't find a workaround at the NativeAOT compilation level, so I had to look for another way.

# Fixing the initialization - the sane way

During my first attempts, I use a simple workaround around the `GC_Initialize` issue. Since NativeAOT crashes when you export a function with that name, the fix is simply to rename it to something else. I renamed it to `Custom_GC_Initialize`, then wrote a small C++ wrapper that calls the renamed function:

```cpp
#include "pch.h"

#include <windows.h>
#include <stdlib.h>

typedef HRESULT(__stdcall* f_GC_Initialize)(void*, void*, void*, void*);
typedef HRESULT(__stdcall* f_GC_VersionInfo)(void*);

static f_GC_Initialize s_gcInitialize;
static f_GC_VersionInfo s_gcVersionInfo;

BOOL APIENTRY DllMain(HMODULE hModule,
	DWORD  ul_reason_for_call,
	LPVOID lpReserved
)
{
	if (ul_reason_for_call == DLL_PROCESS_ATTACH)
	{
		auto module = LoadLibraryA("ManagedDotnetGC.dll");

		if (module == 0)
		{
			return FALSE;
		}

		s_gcInitialize = (f_GC_Initialize)GetProcAddress(module, "Custom_GC_Initialize");
		s_gcVersionInfo = (f_GC_VersionInfo)GetProcAddress(module, "Custom_GC_VersionInfo");
	}

	return true;
}

extern "C" __declspec(dllexport) HRESULT GC_Initialize(
	void* clrToGC,
	void** gcHeap,
	void** gcHandleManager,
	void* gcDacVars
)
{
	return s_gcInitialize(clrToGC, gcHeap, gcHandleManager, gcDacVars);
}

extern "C" __declspec(dllexport) void GC_VersionInfo(void* result)
{
	s_gcVersionInfo(result);
}
```

This code is a simple DLL that loads the original `ManagedDotnetGC.dll`, retrieves the `Custom_GC_Initialize` and `Custom_GC_VersionInfo` functions, then exposes them as `GC_Initialize` and `GC_VersionInfo`. It means that the .NET application has to use the loader as custom GC, and the loader will forward the calls to the actual custom GC.

It works, and it's honestly quite clean, but it bothered me because the goal was to write a custom GC **entirely** in C#. So I carefully reviewed the standalone GC loading code and found a flaw I could exploit.

# Fixing the initialization - the devious way

To fix the problem, we need to stop NativeAOT from calling our `GC_Initialize` function. The only way I found is to enable the aforementioned standalone GC support in NativeAOT by adding a property in the csproj:

```xml
<PropertyGroup>
    <IlcStandaloneGCSupport>true</IlcStandaloneGCSupport>
</PropertyGroup>
```

From there, we can set the `DOTNET_GCName` environment variable and point to the original .NET GC (`clrgc.dll`). This way, NativeAOT will call `GC_Initialize` on that GC instead of ours. However, environment variables are set at the process level, so this causes the test application to also load the original GC instead of the custom one. So we need a way to somehow tell the NativeAOT runtime to use the original GC, and the .NET runtime to use our custom GC.

I got closer to the goal when I realized an interesting quirk: .NET supports environment variables prefixed by either `DOTNET_` or `COMPlus_`, whereas NativeAOT only supports `COMPlus_`. So if we set `COMPlus_GCName=ManagedDotnetGC.dll`, only the .NET runtime will pick it up, and the NativeAOT runtime will ignore it. Therefore, I considered doing this:
```bash
set COMPlus_GCName=ManagedDotnetGC.dll
set DOTNET_GCName=clrgc.dll
```

Unfortunately, `DOTNET_` has a higher priority than `COMPlus_`, so setting both `DOTNET_GCName` and `COMPlus_GCName` causes the .NET runtime to ignore `COMPlus_GCName`. I needed one final trick.

The final trick is that `GCPath` has a higher priority than `GCName`. So if you set, for instance:

```bash
set DOTNET_GCPath=gc1.dll
set DOTNET_GCName=gc2.dll
```

Then .NET will load `gc1.dll`. Putting everything together, I ended up with this:

```bash
set DOTNET_GCName=clrgc.dll
set COMPlus_GCPath=ManagedDotnetGC.dll
```

The .NET runtime will ignore `DOTNET_GCName` because `GCPath` (and therefore `COMPlus_GCPath`) has a higher priority. And the NativeAOT runtime will ignore `COMPlus_GCPath` because it only supports the `DOTNET_` prefix. We end up with the result that we wanted: the .NET runtime uses our custom GC, and the NativeAOT runtime uses the original GC.

For this to work, keep in mind that we need to include `clrgc.dll` along with our custom GC. The file can be found in the .NET installation folder, in `shared\Microsoft.NETCore.App\9.x.x`.

# Wrapping it up

After using either workaround, we can see that the GC is properly loaded when running the test application:

{{<image classes="fancybox center" src="/images/2025-28-01-writing-a-net-gc-in-c-part-1-4.png" >}}

The application does not actually start because we haven't actually implemented the initialization of our custom GC, but that's something we will deal with next time.

The code of this article is available on [GitHub](https://github.com/kevingosse/ManagedDotnetGC/tree/Part1).

{{<rawhtml>}}
<a href="https://amzn.to/42tg58c" target="_blank" rel="noopener noreferrer" style="text-decoration: none; color: inherit;">
  <div style="display: flex; align-items: center; border: 2px solid #ccc; border-radius: 8px; padding: 10px; background-color: #f9f9f9; box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1); margin-top: 50px;">
    <span style="margin-right: 15px; font-size: 1.1em;">
      Liked this article? Don't hesitate to check the 2nd edition of <strong><u>Pro .NET Memory Management</u></strong> for more insights on the .NET Garbage Collector internals!
    </span>
    <img src="/images/progc.png" alt="Pro .NET Memory Management" style="width: 100px; height: auto; border-radius: 4px;" />
  </div>
</a>
{{</rawhtml>}}
