---
url: finding-an-instance-of-thread-by-id-in-windbg-d9fd9a4f1bac
canonical_url: https://medium.com/@kevingosse/finding-an-instance-of-thread-by-id-in-windbg-d9fd9a4f1bac
title: Finding an instance of thread by id in WinDbg
subtitle: Finding an instance of thread by id in WinDbg, with a .NET Core memory dump
date: 2021-09-16
description: ""
tags:
- dotnet
- dotnet-core
- programming
- debugging
- windbg
author: Kevin Gosse
---

As you may already know, it's possible to list all the managed threads in a .NET memory dump using the `!threads` command:

```
0:000> !threads
ThreadCount:      21
UnstartedThread:  0
BackgroundThread: 3
PendingThread:    0
DeadThread:       0
Hosted Runtime:   no
                                                                                                            Lock  
 DBG   ID     OSID ThreadOBJ           State GC Mode     GC Alloc Context                  Domain           Count Apt Exception
   0    1     359c 000001C5340BDFB0    2a020 Preemptive  000001C53439A3F8:000001C53439C000 000001c53225e5b0 -00001 MTA 
   6    2     3e1c 000001C5340E7C00    2b220 Preemptive  0000000000000000:0000000000000000 000001c53225e5b0 -00001 MTA (Finalizer) 
   7    3     5a4c 000001C5340ECF20  102a220 Preemptive  0000000000000000:0000000000000000 000001c53225e5b0 -00001 MTA (Threadpool Worker) 
   8    4     1940 000001C534165830  202b020 Preemptive  000001C53436DEC8:000001C53436F388 000001c53225e5b0 -00001 MTA 
   9    5     7214 000001C534166E60  202b020 Preemptive  000001C53436F4E0:000001C534371388 000001c53225e5b0 -00001 MTA 
  10    6     7034 000001C534169C60  202b020 Preemptive  000001C534371588:000001C534371FD0 000001c53225e5b0 -00001 MTA 
  11    7     6e38 000001C53416C720  202b020 Preemptive  000001C534372448:000001C534373FD0 000001c53225e5b0 -00001 MTA 
  12    8     7220 000001C534170E90  202b020 Preemptive  000001C534374348:000001C534375FD0 000001c53225e5b0 -00001 MTA 
  13    9     5684 000001C534173570  202b020 Preemptive  000001C534376410:000001C534377FD0 000001c53225e5b0 -00001 MTA 
  14   10     4364 000001C534179330  202b020 Preemptive  000001C5343784C8:000001C534379FD0 000001c53225e5b0 -00001 MTA 
  15   11     63ec 000001C534178D20  202b020 Preemptive  000001C53437A688:000001C53437BFD0 000001c53225e5b0 -00001 MTA 
  16   12     1970 000001C534175CA0  202b020 Preemptive  000001C53437C6B8:000001C53437DFD0 000001c53225e5b0 -00001 MTA 
  17   13     6bcc 000001C5341768C0  202b020 Preemptive  000001C53437E7C8:000001C53437FFD0 000001c53225e5b0 -00001 MTA 
  18   14     6890 000001C534178100  202b020 Preemptive  000001C534380908:000001C534381FD0 000001c53225e5b0 -00001 MTA 
  19   15     69d0 000001C5341762B0  202b020 Preemptive  000001C534382A38:000001C534383FD0 000001c53225e5b0 -00001 MTA 
  20   16     1dc8 000001C534178710  202b020 Preemptive  000001C534384B78:000001C534385FD0 000001c53225e5b0 -00001 MTA 
  21   17     1db4 000001C534176ED0  202b020 Preemptive  000001C534386CC8:000001C534387FD0 000001c53225e5b0 -00001 MTA 
  22   18     6aa8 000001C5341774E0  202b020 Preemptive  000001C534388E28:000001C534389FD0 000001c53225e5b0 -00001 MTA 
  23   19     6b94 000001C534177AF0  202b020 Preemptive  000001C53438AF98:000001C53438BFD0 000001c53225e5b0 -00001 MTA 
  24   20      628 000001C534188A20  202b020 Preemptive  000001C53438D490:000001C53438DFD0 000001c53225e5b0 -00001 MTA 
  25   21     66d8 000001C534187E00  1029220 Preemptive  0000000000000000:0000000000000000 000001c53225e5b0 -00001 MTA (Threadpool Worker) 
```

The "ID" column gives you the managed thread id, which is the same value that you could retrieve from the code by calling `thread.ManagedThreadId`. Let's say we are interested by the thread with the managed id "16".

You can switch the active thread by retrieving the value of the "DBG" column (so for the thread with managed id 16, that would be "20") and giving it to the `~` command:

```
0:000> ~20s
ntdll!NtDelayExecution+0x14:
00007ffc`c7f4d3f4 c3              ret

0:020> 
```

When switching thread, WinDbg shows the top frame of the native callstack (here, `ntdll!NtDelayExecution+0x14`) and changes the prompt to `0:020>` to show that thread 20 is now the active thread.

But what if you're interesting in getting the instance of `System.Threading.Thread` associated to this thread?

The fourth column of the output of `!threads` is named "ThreadOBJ" and contains a value that looks like a pointer. From there, you might be tempted to think this is the address of the thread object, but unfortunately...

```
0:020> !threads
ThreadCount:      21
UnstartedThread:  0
BackgroundThread: 3
PendingThread:    0
DeadThread:       0
Hosted Runtime:   no
                                                                                                            Lock  
 DBG   ID     OSID ThreadOBJ           State GC Mode     GC Alloc Context                  Domain           Count Apt Exception
[...] 
  20   16     1dc8 000001C534178710  202b020 Preemptive  000001C534384B78:000001C534385FD0 000001c53225e5b0 -00001 MTA
[...]

0:020> !do 000001C534178710
<Note: this object has an invalid CLASS field>
Invalid object
```

... this is the address of the native thread object, not the managed one.

So how can we retrieve the address of the managed thread? We could enumerate all the threads with `!dumpheap -mt` and inspect all of them individually until finding the right one, but that's going to be really tedious if you have tens, or hundreds of threads. Wouldn't that be great if we could just call `Thread.CurrentThread` from WinDbg?

Well it turns out we kind of can. Following [an optimization in .NET Core 3.0](https://github.com/dotnet/coreclr/pull/21328), the value of `Thread.CurrentThread` is now stored in a thread-static field.

To retrieve the value, we first need to retrieve the EEClass address:

```
0:020> !name2ee System.Private.CoreLib.dll System.Threading.Thread
Module:      00007ffb8e704020
Assembly:    System.Private.CoreLib.dll
Token:       000000000200026D
MethodTable: 00007ffb8e8ca8c0
EEClass:     00007ffb8e8b67d0
Name:        System.Threading.Thread
```

We can then give that value to the `!dumpclass` command (or click on the DML link next to `EEClass:` ):

```
0:020> !dumpclass 00007ffb8e8b67d0
Class Name:      System.Threading.Thread
mdToken:         000000000200026D
File:            C:\Program Files\dotnet\shared\Microsoft.NETCore.App\5.0.10\System.Private.CoreLib.dll
Parent Class:    00007ffb8e8b6748
Module:          00007ffb8e704020
Method Table:    00007ffb8e8ca8c0
Vtable Slots:    4
Total Method Slots:  6
Class Attributes:    100101  
NumInstanceFields:   8
NumStaticFields:     4
NumThreadStaticFields: 1
              MT    Field   Offset                 Type VT     Attr            Value Name
00007ffb8e91c1e8  40009e6        8 ....ExecutionContext  0 instance           _executionContext
0000000000000000  40009e7       10 ...ronizationContext  0 instance           _synchronizationContext
00007ffb8e8c7a90  40009e8       18        System.String  0 instance           _name
00007ffb8e8c1a30  40009e9       20      System.Delegate  0 instance           _delegate
00007ffb8e800c68  40009ea       28        System.Object  0 instance           _threadStartArg
00007ffb8e80ee60  40009eb       30        System.IntPtr  1 instance           _DONT_USE_InternalThread
00007ffb8e80b258  40009ec       38         System.Int32  1 instance           _priority
00007ffb8e80b258  40009ed       3c         System.Int32  1 instance           _managedThreadId
00007ffb8e80b258  40009ef      914         System.Int32  1   static                7 s_optimalMaxSpinWaitsPerSpinIteration
00007ffb8e807238  40009f0      918       System.Boolean  1   static                0 s_isProcessorNumberReallyFast
0000000000000000  40009f1      760                       0   static 0000000000000000 s_asyncLocalPrincipal
00007ffb8e8ca8c0  40009f2       18 ....Threading.Thread  0 TLstatic  t_currentThread
    >> Thread:Value 359c:000001c53436bf88 1940:000001c53436be40 7214:000001c53436c050 7034:000001c53436c128 6e38:000001c53436c200 7220:000001c53436c2d8 5684:000001c53436c3b0 4364:000001c53436c488 63ec:000001c53436c560 1970:000001c53436c638 6bcc:000001c53436c710 6890:000001c53436c7e8 69d0:000001c53436c8c0 1dc8:000001c53436c998 1db4:000001c53436ca70 6aa8:000001c53436cb48 6b94:000001c53436cc20 628:000001c53436ccf8 <<
```

The last line (the one after the `t_currentThread` field) gives the value of the field for each thread. The id used is the "OS ID" given by the `!threads` command:

```
0:020> !threads
ThreadCount:      21
UnstartedThread:  0
BackgroundThread: 3
PendingThread:    0
DeadThread:       0
Hosted Runtime:   no
                                                                                                            Lock  
 DBG   ID     OSID ThreadOBJ           State GC Mode     GC Alloc Context                  Domain           Count Apt Exception
[...] 
  20   16     1dc8 000001C534178710  202b020 Preemptive  000001C534384B78:000001C534385FD0 000001c53225e5b0 -00001 MTA
[...]
```

So in our case we just need to find the value prefixed by `1dc8` :

```
>> Thread:Value 359c:000001c53436bf88 1940:000001c53436be40 7214:000001c53436c050 7034:000001c53436c128 6e38:000001c53436c200 7220:000001c53436c2d8 5684:000001c53436c3b0 4364:000001c53436c488 63ec:000001c53436c560 1970:000001c53436c638 6bcc:000001c53436c710 6890:000001c53436c7e8 69d0:000001c53436c8c0 1dc8:000001c53436c998 1db4:000001c53436ca70 6aa8:000001c53436cb48 6b94:000001c53436cc20 628:000001c53436ccf8 <<
```

Then we can use the `!dumpobj` command to confirm that this is indeed the address of the instance of `System.Threading.Thread` associated to the thread with managed id 16:

```
0:020> !do 000001c53436c998
Name:        System.Threading.Thread
MethodTable: 00007ffb8e8ca8c0
EEClass:     00007ffb8e8b67d0
Size:        72(0x48) bytes
File:        C:\Program Files\dotnet\shared\Microsoft.NETCore.App\5.0.10\System.Private.CoreLib.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
[...]
00007ffb8e80b258  40009ed       3c         System.Int32  1 instance               16 _managedThreadId
[...]
```

Of course this won't work if you're using a version of .NET older than 3.0. In that case, well, I hope you like [scripting in WinDbg](https://stackoverflow.com/a/4616882/869621) 😉


