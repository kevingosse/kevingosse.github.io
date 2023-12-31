---
url: another-way-to-use-your-debugging-tools-7e7f498d7a2b
canonical_url: https://medium.com/@kevingosse/another-way-to-use-your-debugging-tools-7e7f498d7a2b
title: Another way to use your debugging tools
subtitle: It's Friday! And when it's Friday and your computer is managed by Criteo,
  you have a chance to get this nice popup.
date: 2020-02-21
description: ""
tags:
- dotnet
- debugging
- fun
author: Kevin Gosse
thumbnailImage: /images/another-way-to-use-your-debugging-tools-7e7f498d7a2b-1.webp
---

It's Friday! And when it's Friday and your computer is managed by Criteo, you have a chance to get this nice popup:

{{<image classes="fancybox center" src="/images/another-way-to-use-your-debugging-tools-7e7f498d7a2b-1.webp" title="Your computer is about to restart. This action will have consequences." >}}

It's really nice to warn me that my computer is going to restart in a few hours. But it would be even better if the "Hide" button wasn't grayed out. To make things better, the close button doesn't work either, and the window is top-most. So the only choices you have is either to restart now or to spend the Friday with this window eating your precious screen real estate.

Or maybe there's a third choice? I mean, it's Friday after all.

# Sorry boss, I'm working

Checking with ProcessExplorer, we can see that the window is displayed by `SCNotification.exe`, itself launched by `CcmExec.exe`:

{{<image classes="fancybox center" src="/images/another-way-to-use-your-debugging-tools-7e7f498d7a2b-2.webp" >}}

The yellow color indicates that SCNotification is a .net application.

CcmExec is the process associated to the "SMS Agent Host" service.

{{<image classes="fancybox center" src="/images/another-way-to-use-your-debugging-tools-7e7f498d7a2b-3.webp" >}}

I don't know what it actually manages, so I don't want to disable it. Killing `SCNotification.exe` brings a bit of peace but the notification is displayed again after just a few seconds.

I've tried to be more sneaky and replace SCNotification.exe with a dummy exe and it worked for a few days but then the original file was automatically restored somehow. Of course I could replace with a dummy exe **and** remove the write permissions from the file, but maybe there's a more diplomatic way to get some peace?

We've seen earlier that it's a .net application. How is it working? To find out, I fired up a decompiler and started digging into SCNotification.

The entry point is in `SingleInstanceApplication`, which inherits from `System.Windows.Application`. Looks like it's a WPF application.

{{<image classes="fancybox center" src="/images/another-way-to-use-your-debugging-tools-7e7f498d7a2b-9.webp" >}}

I was greeted by a lot of WMI and [CAS](https://docs.microsoft.com/en-us/dotnet/framework/misc/code-access-security-basics?WT.mc_id=DT-MVP-5003493) code. I really didn't want to dig into that (who'd want to?), so I tried to locate where the "remaining before your computer restarts automatically" string was defined. I couldn't find anything, so I went back to ProcessExplorer to check what other assemblies were loaded by the process:

{{<image classes="fancybox center" src="/images/another-way-to-use-your-debugging-tools-7e7f498d7a2b-4.webp" >}}

SCClient.Pages turned out to be the right one. The name of the nagging window was `RestartCoutdownDialog`. I started checking the code to understand why the "Hide" button was grayed out, and quickly found the culprit. The dialog is initialized with two parameters: `countdown` and `countdownAlert`. The former indicates how much time is left before the computer restarts, the latter is the threshold after which the "Hide" button becomes disabled. Every second, a timer ticks and the application checks if the remaining time is below `countdownAlert`:

```csharp
if (this.remainingSeconds <= this.countdownAlert && this.buttonHide.IsEnabled)
{
  Microsoft.SoftwareCenter.Client.Data.Log.Verbose("RestartCountdownDialog: in final countdown, show dialog and do not allow hide/close");
  this.buttonHide.IsEnabled = false;
  this.Show();
}
```

The initial value of `countdownAlert` is fetched from WMI, so it doesn't look like something that can be easily changed.

Is there something else I can do? Modifying the assembly is tempting, but we've already seen that it will be automatically restored after a while. Instead, we can just take advantage of the fact the app is written in WPF.

In Visual Studio, go into Tools → Options → Debugging, and make sure "Enable UI Debugging Tools for XAML" is checked.

{{<image classes="fancybox center" src="/images/another-way-to-use-your-debugging-tools-7e7f498d7a2b-5.webp" >}}

Then go into "Debug → Attach to process" and attach to the SCNotification process. The UI for the XAML debugging tools should appear:

{{<image classes="fancybox center" src="/images/another-way-to-use-your-debugging-tools-7e7f498d7a2b-6.webp" title="Just you wait..." >}}

In Visual Studio, make sure the Live Visual Tree is open (Debug → Windows → Live Visual Tree). You should see the window and the controls it hosts:

{{<image classes="fancybox center" src="/images/another-way-to-use-your-debugging-tools-7e7f498d7a2b-7.webp" >}}

Select the dialog, right click on it, and select "Show properties". This should open the "Live Property Explorer" panel. Search the "Visibility" dropdown and just change it to "Collapsed"!

{{<image classes="fancybox center" src="/images/another-way-to-use-your-debugging-tools-7e7f498d7a2b-8.webp" >}}

Now you can detach your debugger and look for other excuses to procrastinate.
