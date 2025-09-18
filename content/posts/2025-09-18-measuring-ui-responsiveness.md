---
url: measuring-ui-responsiveness
title: Measuring UI responsiveness in Resharper
subtitle: A walkthrough of how I built a custom profiler to measure UI responsiveness, using .NET and Silhouette.
summary: A walkthrough of how I built a custom profiler to measure UI responsiveness, using .NET and Silhouette.
date: 2025-09-18
tags:
- dotnet
- nativeaot
- profiler
- performance
author: Kevin Gosse
thumbnailImage: /images/2025-09-18-measuring-ui-responsiveness-1.jpg
---

_If you're interested in learning more about the performance improvements in the latest iteration of ReSharper, [check this article on the JetBrains blog](https://blog.jetbrains.com/dotnet/2025/08/28/resharper-s-new-out-of-process-engine-cuts-ui-freezes-in-visual-studio-by-80/)._

If you follow me on social media, you probably know that I recently moved to JetBrains, where I work on fixing the performance of ReSharper. Because we can't possibly fix everything at once, we decided to prioritize two areas, based on user feedback: responsiveness/typing latency, and startup time. The vision is to make ReSharper's most used features (like navigation) available as soon as possible, while loading the others in the background. But for this to make sense, we must ensure that the user can use Visual Studio with minimal interference from the background loading. Hence the need to start measuring and putting numbers on _UI responsiveness_.

To do so, I built a tool that continuously measures how long it takes for the main thread to process pending inputs. I wanted something very visual, so I could use Visual Studio normally while getting instant feedback about how responsive the IDE is. I achieved this by adding an overlay, directly inspired from FPS counters used to benchmark games.

{{<video src="https://blog.jetbrains.com/wp-content/uploads/2025/08/Resharper-Sidebyside.mp4" preload="auto">}}

To make it possible, I needed a way to monitor what the UI thread is doing. One way to do so is to use ETW. The `Microsoft-Windows-Win32k` provider has a `UIUnresponsiveness` message that is sent when the window stops pumping messages, so it's possible to build a listener using the [TraceEvent](https://www.nuget.org/packages/Microsoft.Diagnostics.Tracing.TraceEvent) library. For reference, this is how UI freezes were measured [in this JetBrains blog article about ReSharper out-of-process mode](https://blog.jetbrains.com/dotnet/2025/04/01/resharper-out-of-process-update/). There were however two things I didn't like about this approach. First, Windows uses a fixed threshold to consider that a window isn't responsive, and that threshold can't be modified. In my tests, it seems to be around 250 ms, which is too long for my taste. Ideally, I would like to detect pauses of 100 ms or even lower. Second, the message is sent at the end of the freeze, when the window becomes responsive again. This is not a problem for a synthetic benchmark, but not great for a real-time visualization. So instead, I decided to write a custom profiler.

# The UI Profiler

Because I would go to great lengths to avoid writing C++, I built a .NET library to write profilers in C#, named [Silhouette](https://github.com/kevingosse/silhouette/). I already [wrote about it](https://minidump.net/writing-a-net-profiler-in-c-part-5/), so I won't dwell on it too much.

I started by creating a solution with two projects: a C# ClassLibrary project with NativeAOT for the profiler, and a WPF application for the overlay. In the profiler, I added the [Silhouette nuget package](https://www.nuget.org/packages/Silhouette), then I declared a `DllMain` class exposing a `DllGetClassObject` function that is called by .NET when loading the profiler. Because the profiler is loaded through environment variables, and because I didn't want my profiler to be injected into every subprocess spawned by Visual Studio, I added some code to filter by process name and override the environment variables:

```csharp
internal class DllMain
{
    private static ClassFactory? _instance;

    [UnmanagedCallersOnly(EntryPoint = "DllGetClassObject")]
    public static unsafe HResult DllGetClassObject(Guid* rclsid, Guid* riid, nint* ppv)
    {
        if (!string.Equals(Process.GetCurrentProcess().ProcessName, "devenv", StringComparison.OrdinalIgnoreCase))
        {
            return unchecked((int)0x80131375); /* CORPROF_E_PROFILER_CANCEL_ACTIVATION */
        }

        if (*rclsid != new Guid("0A96F866-D763-4099-8E4E-ED1801BE9FBD"))
        {
            Logger.Log($"DllGetClassObject: Invalid CLSID {*rclsid}");
            return HResult.E_NOINTERFACE;
        }

        // Disable profiling for any child processes
        Environment.SetEnvironmentVariable("COR_ENABLE_PROFILING", "0");

        _instance = new ClassFactory(new CorProfilerCallback());
        *ppv = _instance.IClassFactory;

        Logger.Log("Profiler initialized");

        return HResult.S_OK;
    }
}
```

The next question was: how to measure the activity of the UI thread? My first approach was to use the [`SendMessage`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendmessage?WT.mc_id=DT-MVP-5003493) API, which synchronously posts a message to the window and waits for it to be processed. I would then just have to measure how long it takes for the call to complete. It seemed to work at first, but after further testing I discovered that it failed to detect many freezes. The reason is that window messages actually have priorities, and `SendMessage` is executed with the highest priority. Imagine that the window is receiving a constant stream of messages posted with `SendMessage`: those messages will be processed as they arrive so it may look like the UI thread is responsive. However, the low-priority input messages will stay indefinitely at the end of the queue, forever preempted by a new `SendMessage`, and so the UI is not actually responding to user input.
I then experimented with timers, using [`SetTimer`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-settimer?WT.mc_id=DT-MVP-5003493). This solution has the opposite problem: timers run with the lowest priority, even lower than input messages. So when flooding the message queue with input messages, for instance by moving the mouse around, the timer messages would get delayed and the profiler would report that the UI was unresponsive, even though that wasn't the case.
In the end, to get accurate results, the best way was to somehow get notified when input messages are actually processed. Fortunately there is a way: the [`SetWindowsHookEx`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwindowshookexa?WT.mc_id=DT-MVP-5003493) API allows to hook various stages of the message loop, and in particular the `WH_MOUSE` keyword notifies when mouse messages are dequeued. There was one problem still: the user is not going to continuously move their mouse, yet I needed an uninterrupted stream of inputs to detect moments when the UI becomes unresponsive. The solution here was simply to call [`SendInput`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendinput?WT.mc_id=DT-MVP-5003493) in a loop to simulate mouse activity.

The final implementation uses three threads: one to simulate mouse inputs, one to measure the delay between each input, and one to forward the information to the overlay process using named pipes.

The input thread is just an infinite loop, calling `SendInput` every 10 milliseconds. To minimize the impact on the user workflow, the mouse moves back and forth by 2 pixels. I initially tried 1 pixel but I got inconsistent results, with inputs getting ignored. I assume it has something to do with DPI settings.

```csharp
    private void InputThread()
    {
        int mouseDelta = 2;

        try
        {
            var inputs = new NativeMethods.INPUT[1];

            while (true)
            {
                Thread.Sleep(10);                

                inputs[0] = new()
                {
                    mi = new()
                    {
                        dx = mouseDelta,
                        dy = 0,
                        dwFlags = 0x1 /* MOUSEEVENTF_MOVE */
                    },
                    type = 0x0 /* INPUT_MOUSE */
                };

                mouseDelta = -mouseDelta;

                var res = NativeMethods.SendInput(1, inputs, Marshal.SizeOf<NativeMethods.INPUT>());

                if (res != 1)
                {
                    var error = Marshal.GetLastWin32Error();
                    Logger.Log($"SendInput returned {res} - {error:x2}");
                }
            }
        }
        catch (Exception ex)
        {
            Logger.Log($"InputThread failed: {ex}");
        }
    }
```

To hook the message loop, I call `SetWindowsHookEx`, giving it a managed callback. In the callback, I set a mutex that will be watched by another thread:

```csharp
    private readonly ManualResetEventSlim _responsiveMutex = new(false);

    private void SetHook(int threadId)
    {
        NativeMethods.SetWindowsHookEx(NativeMethods.HookType.WH_MOUSE, HookProc, 0, threadId);

        IntPtr HookProc(int code, IntPtr wParam, IntPtr lParam)
        {
            if (code >= 0)
            {
                _responsiveMutex.Set();
            }

            return NativeMethods.CallNextHookEx(0, code, wParam, lParam);
        }
    }
```

The monitoring thread watches the mutex and considers that the UI has become unresponsive if the mutex hasn't been set in 20 milliseconds. When the state changes, a message is enqueued to a `BlockingCollection`, to be sent to the overlay.

```csharp
    private readonly BlockingCollection<string> _messages = new();

    private void MonitoringThread()
    {
        bool isResponsive = true;
        var stopwatch = Stopwatch.StartNew();

        while (true)
        {
            if (_responsiveMutex.Wait(20))
            {
                _responsiveMutex.Reset();

                if (!isResponsive)
                {
                    isResponsive = true;
                    stopwatch.Stop();
                    _messages.Add($"true|{stopwatch.ElapsedMilliseconds}");
                }
            }
            else
            {
                if (isResponsive)
                {
                    isResponsive = false;

                    stopwatch.Restart();
                    _messages.Add("false");

                    // Now wait indefinitely
                    _responsiveMutex.Wait();
                }
            }
        }
    }
```

{{< alert >}}
In hindsight, the design might seem odd, and is the result of iterating over multiple failed solutions. If I had to start over, I believe I would simply send a message to the overlay every time the window hook is called, and let the overlay deduce whether the UI is responsive or not. In other words, I would move the logic of this thread to the overlay.
{{< /alert >}}

The last thread handles the communication with the overlay. It launches the overlay process, then simply dequeues the messages from the `BlockingCollection` and sends them over named pipes:

```csharp
    private void SenderThread()
    {
        try
        {
            var pipeName = $"UIProfiler-{Guid.NewGuid()}";

            var startInfo = new ProcessStartInfo(_overlayPath)
            {
                UseShellExecute = false,
                Arguments = $"{Environment.ProcessId} {pipeName}"
            };

            Process.Start(startInfo);

            using var pipeClient = new NamedPipeClientStream(".", pipeName, PipeDirection.InOut);
            pipeClient.Connect();

            using var writer = new StreamWriter(pipeClient);
            writer.AutoFlush = true;

            foreach (var message in _messages.GetConsumingEnumerable())
            {
                writer.WriteLine(message);
            }
        }
        catch (Exception ex)
        {
            Logger.Log($"SenderThread failed: {ex}");
        }
    }
```

# The overlay

The role of the overlay is to listen to the UI responsiveness information from the named pipe, and display it on top of Visual Studio's window. It's a small WPF application with a few subtleties. It uses [LiveChartsCore](https://www.nuget.org/packages/LiveChartsCore/) to display the graph.

The first subtlety is how to locate the Visual Studio window, to display the overlay on top. It receives the Visual Studio pid as an argument at startup, then it iterates over all windows of that process until it finds one with the caption "Visual Studio Preview" (to skip the splashscreen).

{{< alert >}}
Ultimately, displaying the overlay over the splashscreen wouldn't be an issue, but I was too lazy to handle resizing so I wanted to directly target the main window.
{{< /alert >}}

```csharp
    private IntPtr FindVsWindow(int pid)
    {
        while (true)
        {
            var mainWindow = FindWindowWithCaption((uint)pid, "Visual Studio Preview");

            if (mainWindow != IntPtr.Zero)
            {
                return mainWindow;
            }

            // The main window wasn't found, try again in a while
            Thread.Sleep(100);
        }
    }

    private static IntPtr FindWindowWithCaption(uint processId, string windowCaption)
    {
        var foundWindow = IntPtr.Zero;

        // Enumerate all top-level windows
        NativeMethods.EnumWindows(EnumWindowsCallback, IntPtr.Zero);

        return foundWindow;

        bool EnumWindowsCallback(IntPtr hWnd, IntPtr lParam)
        {
            // Filter by process ID
            NativeMethods.GetWindowThreadProcessId(hWnd, out uint windowProcessId);

            if (windowProcessId != processId)
            {
                return true; // Skip this window (continue enumeration)
            }

            // Get the window's title/caption
            var windowTitle = new StringBuilder(256);
            NativeMethods.GetWindowText(hWnd, windowTitle, 256);

            // Check if the window is visible and matches the desired caption
            if (NativeMethods.IsWindowVisible(hWnd) && windowTitle.ToString().Contains(windowCaption, StringComparison.OrdinalIgnoreCase))
            {
                foundWindow = hWnd; // We found the window
                return false;       // Stop enumeration
            }

            return true; // Continue enumeration
        }
    }
```

Another subtlety is setting the `WS_EX_TRANSPARENT` style on the window. It allows to display the overlay on top of the Visual Studio window without stealing its inputs. As far as I know, WPF doesn't have an API to set it, so I had to use some interop:

```csharp
    private static OverlayWindow LoadOverlay()
    {
        var overlay = new OverlayWindow
        {
            ShowInTaskbar = false,
            ShowActivated = false,
            Topmost = true
        };

        overlay.SourceInitialized += (s, e) =>
        {
            var handle = new WindowInteropHelper(overlay).Handle;
            var extendedStyle = NativeMethods.GetWindowLong(handle, NativeMethods.GWL_EXSTYLE);

            // Add WS_EX_TRANSPARENT style to allow clicks to pass through
            _ = NativeMethods.SetWindowLong(
                handle,
                NativeMethods.GWL_EXSTYLE,
                extendedStyle | NativeMethods.WS_EX_TRANSPARENT);
        };

        return overlay;
    }
```

The last subtle bit is to properly size the overlay. I needed to get the size of the Visual Studio window, but also to account for DPIs:

```csharp
    public void Show(nint window)
    {
        // Get bounding rectangle in device coordinates
        var hr = NativeMethods.DwmGetWindowAttribute(
            window,
            NativeMethods.DWMWA_EXTENDED_FRAME_BOUNDS,
            out NativeMethods.RECT rect,
            Marshal.SizeOf(typeof(NativeMethods.RECT)));

        if (hr != 0)
        {
            return;
        }

        int windowWidthPx = rect.Right - rect.Left;
        int windowHeightPx = rect.Bottom - rect.Top;

        if (windowWidthPx <= 0 || windowHeightPx <= 0)
        {
            return;
        }

        var dpi = NativeMethods.GetDpiForWindow(window);

        // Convert device pixels -> WPF device-independent pixels (DIPs)
        // 1 DIP = 1 px at 96 DPI. So the scale factor is (96 / actualDPI)
        double scale = 96.0 / dpi;

        Left = rect.Left * scale;
        Top = rect.Top * scale;
        Width = windowWidthPx * scale;
        Height = windowHeightPx * scale;

        Show();
    }
```

And that's it! I won't go over the LiveChartsCore bits, but you can check the full code [on github](https://github.com/kevingosse/UIProfiler).

# Wrapping it up

This was a very educative project, that turned out to be very useful. Measuring is a critical part of any performance optimization work, and the ability to visualize the responsiveness of the UI (instead of just "feeling" it) really helps directing the effort. There are many pain points to fix in this tool (the overlay doesn't follow the VS window if moved or resized, also it mistakenly thinks that the UI is unresponsive if the mouse cursor isn't on top of the window), but I already use it almost daily to evaluate the effects of my optimizations and find new areas that need improvement. For instance, here is me randomly discovering a performance regression that added a significant delay when opening the context menu:
{{<video src="/videos/RightClick.mp4" preload="auto">}}

The problem got swiftly fixed after being discovered.

I'm also very satisfied by how [Silhouette](https://github.com/kevingosse/silhouette/) simplifies the usage of the .NET profiling API. The published version of the UI profiler uses the profiling API only to inject into the target process, but previous iterations also sampled the UI thread to find out what is running during the freezes. I will probably bring back this feature at some point.


