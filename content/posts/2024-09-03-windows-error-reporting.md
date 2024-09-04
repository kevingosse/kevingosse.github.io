---
url: windows-error-reporting
title: Using Windows Error Reporting in .NET
subtitle: A look into how to use Windows Error Reporting to collect crash information for your .NET apps.
summary: A look into how to use Windows Error Reporting to collect crash information for your .NET apps.
date: 2024-09-03
tags:
- dotnet
- nativeaot
author: Kevin Gosse
thumbnailImage: /images/2024-09-03-windows-error-reporting-6.png
---

As I was looking into collecting crash telemetry for Datadog, I started experimenting with WER. WER stands for Windows Error Reporting, and it's an API designed to allow Microsoft and third-party vendors to collect information about... well, errors, and crashes. Unfortunately, this API is very poorly documented, and a large part of the documentation comes from open-source projects (such as crashpad) that had to discover its quirks through trial and error. This article aims to provide a quick-start for .NET developers who would be interested in using WER in their applications.

Note that this article focuses purely on one part of the WER API: the crash handler. There are many more features, such as the ability to manually create a report, that are not covered here.

# How does it work?

First, let's clarify what we're going to do in this article. The goal is to automatically collect information when an application that we own crashes, either to be sent to a telemetry endpoint or just stored in the Windows event log. WER allows to do that through the use of a crash handler. The application has to explicitly register the crash handler (so you can't use this mechanism to arbitrarilty monitor all applications on a machine). Then, when the process crashes, Windows is going to suspend it and spawn the `WerFault.exe` process. `WerFault.exe` will load the registered crash handlers and give them a chance to inspect the memory of the faulty process to collect all the information they need. After that, the process will be teared down like for a normal crash.

This has multiple advantages over a "manual" approach, where you'd try to catch all exceptions in your application. Since the mechanism is handled at the OS level, it supposedly works for all kinds of crashes (whereas, if you try to handle it manually, you may fail to invoke your code in some unrecoverable crash conditions such as a stack overflow). Also, the collection occurs out-of-process, so you don't have to worry about your crash handler getting corrupted by whatever caused the crash.

# Preparing the crash handler

The very first step is to build our crash handler. Normally you'd use C++ for that, but if you follow me then you probably know that I can't resist an occasion to use NativeAOT. So we will build our crash handler in C#.

Create a new "Class Library" project targeting .NET 8+ and make sure to add `<PublishAot>true</PublishAot>` to the csproj (or check the NativeAOT checkbox in Visual Studio). Then we need to export three functions:
- `OutOfProcessExceptionEventCallback`: this is the first function that will be called when the DLL is loaded by `WerFault.exe`. This is where we are given a chance to inspect the crashing process, and we must decide whether we want to "claim" the crash or not. If we claim it, then `WerFault.exe` will call the next two functions.
- `OutOfProcessExceptionEventSignatureCallback`: this function is only called if we claimed the crash during the call to `OutOfProcessExceptionEventCallback`. This is our chance to add custom metadata that will be visible in the crash report in the event log. I believe that information can also be used to triage the crash if you registered to the "Windows Desktop Application Program", but I haven't looked into it.
- `OutOfProcessExceptionEventDebuggerLaunchCallback`: this function is only called if we claimed the crash during the call to `OutOfProcessExceptionEventCallback`. This gives you a chance to launch a custom debugger to debug the crash.

At this point, our code looks like:

```csharp
internal class Wer
{
    [UnmanagedCallersOnly(EntryPoint = "OutOfProcessExceptionEventCallback")]
    private static unsafe int OutOfProcessExceptionEventCallback(nint context, WER_RUNTIME_EXCEPTION_INFORMATION* exceptionInformation, bool* ownershipClaimed, char* eventName, int* size, int* signatureCount)
    {
        Debug.WriteLine("OutOfProcessExceptionEventCallback");
        return 0;
    }

    [UnmanagedCallersOnly(EntryPoint = "OutOfProcessExceptionEventSignatureCallback")]
    private static unsafe int OutOfProcessExceptionEventSignatureCallback(nint context, WER_RUNTIME_EXCEPTION_INFORMATION* exceptionInformation, int index, char* name, int* nameLength, char* value, int* valueLength)
    {
        Debug.WriteLine("OutOfProcessExceptionEventSignatureCallback");
        return 0;
    }

    [UnmanagedCallersOnly(EntryPoint = "OutOfProcessExceptionEventDebuggerLaunchCallback")]
    private static unsafe int OutOfProcessExceptionEventDebuggerLaunchCallback(nint context, WER_RUNTIME_EXCEPTION_INFORMATION* exceptionInformation, int* isCustomDebugger, char* debuggerLaunch, int* debuggerLaunchLength, int* isDebuggerAutoLaunch)
    {
        Debug.WriteLine("OutOfProcessExceptionEventDebuggerLaunchCallback");
        return 0;
    }

    [StructLayout(LayoutKind.Sequential)]
    private unsafe struct EXCEPTION_RECORD
    {
        public const int EXCEPTION_MAXIMUM_PARAMETERS = 15;

        public int ExceptionCode;
        public int ExceptionFlags;
        public EXCEPTION_RECORD* ExceptionRecord;
        public nint ExceptionAddress;
        public int NumberParameters;
        public fixed ulong ExceptionInformation[EXCEPTION_MAXIMUM_PARAMETERS];
    }

    [StructLayout(LayoutKind.Sequential)]
    private unsafe struct WER_RUNTIME_EXCEPTION_INFORMATION
    {
        public int Size;
        public nint Process;
        public nint Thread;
        public EXCEPTION_RECORD ExceptionRecord;
        public nint Context;
        public char* ReportId;
        public int IsFatal;
        public int Reserved;
    }
}

```

In a NativeAOT library, the `UnmanagedCallersOnly` exports the function with the given symbol, in addition to preparing it to be called directly from unmanaged code. Because `WerFault.exe` is not an interactive process, we use `Debug.WriteLine` to write some messages that we can then listen to by using [Sysinternals DebugView](https://learn.microsoft.com/en-us/sysinternals/downloads/debugview) or [Pavel Yosifovich's DbgPrint](https://github.com/zodiacon/AllTools) (which I prefer because it shows the process name).

We can then export the DLL using NativeAOT:
```bash
$ dotnet publish -r win-x64
```

# Registering the crash handler

There are two more steps before we can see our debug messages: registering the crash handler, and... crashing.

The crash handler must first be registered in the registry. A DWORD value must be created, either in `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\Windows Error Reporting\RuntimeExceptionHelperModules` or in `HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\Windows Error Reporting\RuntimeExceptionHelperModules`. The actual value doesn't matter, but the name must be the path to the crash handler DLL (for instance, `C:\crash\wer.dll`).
Ideally the value should be stored in `HKEY_LOCAL_MACHINE`, but this requires administrator permission (so it would typically be something you do during installation). Since Windows 10 20H1, the value can be stored in `HKEY_LOCAL_USER` instead, with one caveat: a crash handler that has been registered in HKCU instead of HKLM can't claim the crash during `OutOfProcessExceptionEventCallback` (so `OutOfProcessExceptionEventSignatureCallback` and `OutOfProcessExceptionEventDebuggerLaunchCallback` won't be called). This should however be enough for most use-cases.

After adding the crash handler to the registry, we still need to make a test application crash. Let's create a new .NET 8 Console Application project (not .NET 9+, for reasons explained later). Inside, we need to call `WerRegisterRuntimeExceptionModule` to register the crash handler. It takes two arguments: the path to the crash handler DLL, and an arbitrary pointer-size value that will be given as-is to the crash handler (so you can use it to pass a custom context). Then, we trigger the crash:

```csharp
class Program
{
    static unsafe void Main(string[] args)
    {
        int result = WerRegisterRuntimeExceptionModule(@"C:\crash\wer.dll", 0);

        if (result != 0)
        {
            Console.WriteLine($"Failed to register crash handler: {result}");
            return;
        }

        Console.WriteLine("Press return to crash");
        Console.ReadLine();

        throw new Exception("Crashing");
    }

    [DllImport("kernel32.dll")]
    public static extern int WerRegisterRuntimeExceptionModule([MarshalAs(UnmanagedType.LPWStr)] string callbackDll, nint context);
}
```

If you did everything properly, you should see the `OutOfProcessExceptionEventCallback` debug message in DbgPrint.

{{<image classes="fancybox center" src="/images/2024-09-03-windows-error-reporting-1.png" >}}

At this point, only `OutOfProcessExceptionEventCallback` is called because we don't claim the crash.

So far, everything seems to work properly, but if you try the same thing in a .NET Framework application (or .NET 9+) you will see that the crash handler is not invoked. Why is that?

# Too many cooks (the part with some debugging)

You may want to skip ahead if you're only interested in WER, but since this is mainly a debugging blog I thought it would be nice to show how to debug WER registration issues.

To check if the handler is properly registered, I started the .NET Framework app (with the same code as the test app shown before) up until the `Press return to crash` point and attached Windbg to the process. From there, you can use `!peb` to dump the Process Environment Block. 

{{<image classes="fancybox center" src="/images/2024-09-03-windows-error-reporting-2.png" >}}

In the output, you should see `PEB at 000000e62f578000` with the address underlined. By clicking on it, we can show the actual fields of the native `_PEB` object (or type the command `dt 0x000000e62f578000 ntdll!_PEB`). Then look for the value of the `WerRegistrationData` field (here, at the offset 0x358).

```
0:006> dt 0x000000e62f578000 ntdll!_PEB
   +0x000 InheritedAddressSpace : 0 ''
   +0x001 ReadImageFileExecOptions : 0 ''
   +0x002 BeingDebugged    : 0x1 ''
   +0x003 BitField         : 0 ''
[...]   
   +0x320 SparePointers    : [4] (null) 
   +0x340 SpareUlongs      : [5] 0
   +0x358 WerRegistrationData : 0x000002bb`c6c90000 Void
   +0x360 WerShipAssertPtr : (null) 
[...]   
```

Open that address in the "Memory" panel in Windbg. In the ribbon, change "Size" to "Long" (or "Integer" if it's a 32-bit process). 

{{<image classes="fancybox center" src="/images/2024-09-03-windows-error-reporting-3.png" >}}

Scroll until you find the `HEAP_SIGNATURE` string, and take the only non-zero value before that string (it should be located 4 pointers before the string). That's the address of the list of registered handlers.

{{<image classes="fancybox center" src="/images/2024-09-03-windows-error-reporting-4.png" >}}

If we inspect the memory at that address, we can see almost immediately the string `C:\Windows\Microsoft.NET\Framework64\v4.0.30319\mscordacwks.dll`, and our own crash handler a bit later (`C:\crash\wer.dll`). It turns out that .NET Framework registers its own crash handler, and they're invoked in the same order they were registered. Because of that, it's able to claim the crash before we have a chance to!

# Changing the order of the crash handlers

Now that we know why our crash handler isn't invoked, what can we do about it? Fortunately there is a `WerUnregisterRuntimeExceptionModule` API that we can use to unregister any crash handler. The plan is to unregister the .NET Framework crash handler, register our own, then re-register the .NET one. This way, we will be called first and get a chance to claim the crash, and the old one will still be called if we decide not to.

There's one difficulty though. If you remember, `WerRegisterRuntimeExceptionModule` takes a second parameter, an arbitrary pointer-sized value used as context. When calling `WerUnregisterRuntimeExceptionModule`, you need to give the exact same value, so we have to figure out what .NET is using.

I often mention it in my articles, you can have a good approximation of the source code of .NET Framework by checking [the first commit of the coreclr repository](https://github.com/dotnet/coreclr/tree/ef1e2ab328087c61a6878c1e84f4fc5d710aebce), before .NET Core diverged too much. Looking into it, [we can see that .NET uses the variable `g_pMSCorEE` as context](https://github.com/dotnet/coreclr/blob/ef1e2ab328087c61a6878c1e84f4fc5d710aebce/src/vm/dwreport.cpp#L255): 

```cpp
HRESULT hr = (*pFnWerRegisterRuntimeExceptionModule)(wszDACPath, (PDWORD)g_pMSCorEE);
```

`g_pMSCorEE` contains the address of the clr.dll module in memory. I assume this is because .NET Framework supports loading multiple versions of .NET side-by-side, so this is used to identify the right runtime.

This information can be retrieved fairly easily, so we can write some code to unregister the .NET crash handler, register our own, then re-register the .NET one:

```csharp
class Program
{
    static unsafe void Main(string[] args)
    {
        // Locate the clr.dll module
        var clr = Process.GetCurrentProcess().Modules.Cast<ProcessModule>().First(m => m.ModuleName == "clr.dll");

        // Build the path to the crash handler (which lives in the DAC)
        var dac = Path.Combine(Path.GetDirectoryName(clr.FileName), "mscordacwks.dll");

        // Unregister the DAC
        var result = WerUnregisterRuntimeExceptionModule(dac, clr.BaseAddress);

        if (result != 0)
        {
            Console.WriteLine($"Failed to unregister {dac} with context {clr.BaseAddress:x2}");
            return;
        }

        // Register our crash handler
        result = WerRegisterRuntimeExceptionModule(@"C:\crash\wer.dll", 0);

        if (result != 0)
        {
            Console.WriteLine($"Failed to register crash handler: {result}");
            return;
        }

        // Re-register the DAC
        result = WerRegisterRuntimeExceptionModule(dac, clr.BaseAddress);

        if (result != 0)
        {
            Console.WriteLine($"Failed to re-register {dac} with context {clr.BaseAddress:x2}");
            return;
        }

        Console.WriteLine("Press return to crash");
        Console.ReadLine();

        throw new Exception("Crashing");
    }

    [DllImport("kernel32.dll")]
    public static extern int WerRegisterRuntimeExceptionModule([MarshalAs(UnmanagedType.LPWStr)] string callbackDll, nint context);

    [DllImport("kernel32.dll")]
    public static extern int WerUnregisterRuntimeExceptionModule([MarshalAs(UnmanagedType.LPWStr)] string callbackDll, nint context);
}
```

And with that, our crash handler should work even on .NET Framework. 

But I mentioned .NET 9+ earlier, what was that about? Since I only needed to call `WerUnregisterRuntimeExceptionModule` on .NET Framework, I initially assumed that the WER crash handler had been removed in .NET Core. However, while debugging an unrelated registration issue in my code (using Windbg as I demonstrated before), I noticed that .NET Core was also registering its own handler! After further inspection, I discovered that the logic in `WerRegisterRuntimeExceptionModule` was broken, so the .NET Core handler never claimed the crash. [I submitted a bug report and it has been fixed](https://github.com/dotnet/runtime/issues/103872), so I expect that the `WerUnregisterRuntimeExceptionModule` workaround will be needed again in future versions of .NET. This is exactly the same logic as for .NET Framework, except that `clr.dll` is replaced with `coreclr.dll`, and `mscordacwks.dll` with `mscordaccore.dll`.

```csharp
        // Locate the coreclr.dll module
        var coreclr = Process.GetCurrentProcess().Modules.Cast<ProcessModule>().First(m => m.ModuleName == "coreclr.dll");

        // Build the path to the crash handler (which lives in the DAC)
        var dac = Path.Combine(Path.GetDirectoryName(coreclr.FileName), "mscordaccore.dll");

        // Unregister the DAC
        var result = WerUnregisterRuntimeExceptionModule(dac, coreclr.BaseAddress);
```

# Implementing a crash handler

Last but not least, we're going to see what kind of logic we can put in the crash handler. You can do all kind of things, especially if you use ClrMD to inspect the memory, but for this article we're only going to see how to use the context argument of `WerRegisterRuntimeExceptionModule` to share some information, and add that to the crash report metadata.

Let's say that we want to store two things in the crash report: a version number, and a status message. It's pretty obvious that all this information is not going to fit into the pointer-sized `context` argument, so instead we will give the address to that information.

First we declare the struct that will store the information. I put the status message in an inline fixed-size string because it makes things easier, but that's not mandatory.

```csharp
[StructLayout(LayoutKind.Sequential)]
struct Payload
{
    public int Version;
    public Message Status;

    [InlineArray(256)]
    public struct Message
    {
        private char _first;
    }
}

```

(we could also declare it as `public fixed char Status[256]`, but I find inline arrays easier to manipulate)

The definition of that struct should be shared between the crashing app and the crash handler.

Then, in the crashing app, we allocate the struct (using some native memory because we don't want it to be moved by the GC) and we store our information:

```csharp
var ptr = NativeMemory.AllocZeroed((nuint)sizeof(Payload));
var payload = (Payload*)ptr;

payload->Version = 5;
"Hello world!".CopyTo(payload->Status);
```

Then we give the address of the struct to `WerRegisterRuntimeExceptionModule`:

```csharp
result = WerRegisterRuntimeExceptionModule(@"C:\crash\wer.dll", (nint)ptr);
```

In `OutOfProcessExceptionEventCallback`, we use `ReadProcessMemory` to read the payload from the crashing process. We can find a handle to the crashing process in the `exceptionInformation` argument. If we successfully read the payload, we claim the crash and tell WER that we want to register two fields in the metadata. As a bonus, we give a custom name to our crash report (don't forget the null terminator!).

```csharp
private static Payload CrashPayload;

[UnmanagedCallersOnly(EntryPoint = "OutOfProcessExceptionEventCallback")]
private static unsafe int OutOfProcessExceptionEventCallback(nint context, WER_RUNTIME_EXCEPTION_INFORMATION* exceptionInformation, bool* ownershipClaimed, char* eventName, int* size, int* signatureCount)
{
    Debug.WriteLine("OutOfProcessExceptionEventCallback");
    Payload payload = default;

    Debug.WriteLine($"Reading memory at {context:x2}");

    var result = ReadProcessMemory(exceptionInformation->Process, context, &payload, (nuint)sizeof(Payload), out var bytesRead);

    if (result && bytesRead == (nuint)sizeof(Payload))
    {
        CrashPayload = payload; // Store the payload for later use
        Debug.WriteLine($"Payload: {payload.Version} {payload.Status}");
        *ownershipClaimed = true;

        var name = "My crash report\0";
        name.CopyTo(new Span<char>(eventName, *size));
        *size = name.Length;

        *signatureCount = 2;
    }
    else
    {
        Debug.WriteLine($"Failed to read memory: {Marshal.GetLastPInvokeError()}");
    }

    return 0;
}

[DllImport("kernel32.dll", SetLastError = true)]
public static extern unsafe bool ReadProcessMemory(nint hProcess, nint lpBaseAddress, void* lpBuffer, nuint nSize, out nuint lpNumberOfBytesRead);
```

Because we claimed the crash and set `signatureCount` to 2, `OutOfProcessExceptionEventSignatureCallback` will be called two times, once for each field.

```csharp
[UnmanagedCallersOnly(EntryPoint = "OutOfProcessExceptionEventSignatureCallback")]
private static unsafe int OutOfProcessExceptionEventSignatureCallback(nint context, WER_RUNTIME_EXCEPTION_INFORMATION* exceptionInformation, int index, char* name, int* nameLength, char* value, int* valueLength)
{
    Debug.WriteLine("OutOfProcessExceptionEventSignatureCallback");

    string signatureKey;
    string signatureValue;

    if (index == 0)
    {
        signatureKey = "Version";
        signatureValue = CrashPayload.Version.ToString();
    }
    else if (index == 1)
    {
        signatureKey = "Status";
        signatureValue = new string(CrashPayload.Status);
    }
    else
    {
        return 0;
    }

    (signatureKey + "\0").CopyTo(new Span<char>(name, *nameLength));
    *nameLength = signatureKey.Length + 1;

    (signatureValue + "\0").CopyTo(new Span<char>(value, *valueLength));
    *valueLength = signatureValue.Length + 1;

    return 0;
}
```

If everything worked, the crash report with the custom metadata will be visible in the event log:

{{<image classes="fancybox center" src="/images/2024-09-03-windows-error-reporting-6.png" >}}

# Wrapping it up

This article barely scratches the surface of what is possible to do with WER. This is a great way to automatically collect crash telemetry for your application, WER removes a lot of the complexity by detecting the crash and running your handler in a separate process. Of course, this is only useful for applications deployed to customers. If running in your own environment, you'll probably want to collect a full memory dump instead, to be sure to have all the required information to diagnose the issue.