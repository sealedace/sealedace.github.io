---
layout: post
title: "Exploring iOS Crash Reports"
date: 2014-03-20 11:25:11 +0800
comments: true
published: false
categories: iOS
---


[original article]: https://www.plausible.coop/blog/?p=176

#####注：本文译自[Exploring iOS Crash Reports][original article]。

##Introduction

作为开发者的我们，当自己开发的app出现crash的时候，我们会收集各种跟crash相关的信息以便于我们定位到crash的原因，最终fix掉（理想的结果）。要获取到crash report，可以通过一些工具还有某些服务，比如iTunes Connect，PLCrashReporter，HockeyApp或者其他一些起初看起来令人生畏的工具或服务。我们这里会慢慢揭开诸如crash report的神秘面纱，在读完这篇文章后，你应该可以更加有效的去阅读处理以及使用crash report。

我们这里主要讨论的是iOS app的crash，尽管如此，一些换汤不换药的概念和方法应该也可以被用到其他平台上去。

当然，有些iOS开发者会关心一些其他原因导致程序的crash，比如watchdog超时，因内存不足导致的应用被kill。这些，我们不在这里讨论，因为早就有人研究过了。

##A First Look
假设我们拿到了线上的app的一个非常完整的crash report，里面包含了大量的信息。我们把它分解成一些容易理解的小的片段，然后分别讨论。


##The Header
在每个crash report的头部，你可以看到一些基本的头信息：

```
Incident Identifier: A8111234-3CD8-4FF0-BD99-CFF7FDACB212
CrashReporter Key: A54D28AF-3010-5839-BBA6-FE72C8AFCC2E
Hardware Model: iPod3,1
Process: OurApp [476]
Path: /Users/USER/OurApp.app/OurApp
Identifier: coop.plausible.OurApp
Version: 48
Code Type: ARM
Parent Process: launchd [1]

Date/Time: 2013-03-30T04:42:07Z
OS Version: iPhone OS 5.1.1 (9B206)
Report Version: 104
```

上面大部分的信息可以从字面上理解是什么意思，但是有几个需要解释下：

* **Incident Identifier**: 客户端为crash report分配的唯一标识。

* **CrashReporter Key**: 这个是客户端分配的，匿名的，并且每一个设备一个的id，有点类似UDID。这个主要为了方便统计该问题的普遍性。
* **Hardware Model**: 这个是出现crash的机器型号。这个主要用于在重现一些只在某些指定机型的机器上出现的问题，不过，这种情况不多。
* **Code Type**: 这个是目标机器的处理器类型。在iOS设备上，这个永远都是`ARM`，不管是`ARMv7`或者`ARMv7s`。
* **OS Version**: crash出现的系统版本，包括版本build编号。这个让我们知道在哪些具体系统版本上会出现crash。需要注意的是，不同型号的iOS设备会被分配一个唯一的build版本号（如：9B206），crash几乎不会只在某一个build版本的系统上出现。
* **Report Version**: 这个不透明的值是Apple给report定义的版本号。当report格式变化了，Apple可能会更新这个版本号。在PLCrashReporter，我们生成并且保存report信息用我们自己定义的基于protobuf数据结构的里，并且可以灵活的兼容Apple的report格式。

##The Stack Trace

在iOS上，一个应用程序是一个典型的独立进程，里面运行着多个线程。其中包括UI主线程，dispatch管理线程，和一些其他线程，类似管理I/O的worker线程等。每个线程都在执行代码，它堆栈跟踪信息会列出这个线程沿着一连串的函数调用一直到当前运行终止的位置。一份完整的crash report，堆栈信息应该包含每一个线程在crash的时刻的状态信息，以及是哪一个线程导致程序crash的。我们从底部向上阅读这个栈信息，栈的顶部是最上层的入口，也就是被调用的最里层的函数。下面的信息显示的是crash出现在了主线程：

```
Thread 0 Crashed:
0 libsystem_kernel.dylib 0x3466e32c ___pthread_kill + 8
1 libsystem_c.dylib 0x3526829f abort + 95
2 OurApp 0x0015dfc3 uncaught_exception_handler + 27
3 CoreFoundation 0x3601b957 __handleUncaughtException + 75
4 libobjc.A.dylib 0x30f91345 __objc_terminate + 129
5 libc++abi.dylib 0x34d333c5 __ZL19safe_handler_callerPFvvE + 77
6 libc++abi.dylib 0x34d33451 __ZdlPv + 1
7 libc++abi.dylib 0x34d34825 ___cxa_current_exception_type + 1
8 libobjc.A.dylib 0x30f912a9 _objc_exception_rethrow + 13
9 CoreFoundation 0x35f7150d _CFRunLoopRunSpecific + 405
10 CoreFoundation 0x35f7136d _CFRunLoopRunInMode + 105
11 GraphicsServices 0x31876439 _GSEventRunModal + 137
12 UIKit 0x3192ecd5 _UIApplicationMain + 1081
13 OurApp 0x000938e7 main (main.m:16)

Thread 1:
...
```

如果我们仔细看堆栈信息，我们会很容易发现导致crash的原因是未处理的异常。（如果你觉得会有很多地方出现，你可以在Xcode里设置一个Exception断点，这样当你调试程序途中抛出异常时，这个断点会告诉你是哪个函数里面的异常。）

可以这样说，大部分开发者在遇到crash时，首先考虑的就是找到crash report，这很好理解。但是经常遇到的情况是，光有堆栈信息是不足以理解潜在的crash原因的。

##Debug Symbols
需要引起注意的是，当程序在开发阶段build的时候，为了减少生成的包的体积，debug symbols会被去掉（debug symbols与编程语言和机器代码相关联）。对于iOS应用，在build完生成二进制文件的同时，旁边会生成一个`.dSYM`文件。保留一份`.dSYM`文件的copy是很重要的，因为即便是同样的一份源文件再次编译，也不一定会重新生成出来。`.dSYM`文件是由`dsymutil`程序生成的，从可执行文件和中间目标文件中制作出来的。（.o文件仍然保留着DWARF的debug信息）

通过二进制文件的`.dSYM`，crash report上得那些符号地址可以重新映射成可读性很高的符号和名字，甚至所在的源代码名字和行号都能看得到。

让我们用两个简短的例子来看下重新符号映射前后的差别。

// 符号映射之前

    8 OurApp 0x000029d4 0x1000 + 6612
第一个列的数字代表这一栈帧在堆栈信息中的索引。第二列代表这个函数名称所属的程序的名称。第三列表示这个被调用函数的地址。最后面的两个部分是这个库的二进制映像的基地址以及偏移量。

可以看到这并不直观，让我们再来看下符号化后的结果：

// 符号映射之后

    8 OurApp 0x000029d4 -[OurAppDelegate applicationDidFinishLaunching:] (OurAppDelegate.m:128)
前面三个字段跟之前的一样，后面实际上就是代码的文件，行号，函数名，很容易理解。

##The Exception Section
异常信息这部分，提供了：异常类型，异常代码，以及出现异常的线程：

	Exception Type: SIGABRT
	Exception Codes: #0 at 0x3466e32c
	Crashed Thread: 0

我们这里谈论的“异常”，不是Objective-C类型的异常（尽管有时候的确是导致crash的原因），而是Mach类型异常。例子中给的异常类型是一个UNIX信号——SIGABRT，这对于很多UNIX程序员来说会比较熟悉。围绕Mach异常和UNIX信号，已经建立了很完整api的生态系统，我们这里不去介绍，如果你有兴趣，可以在最下面*阅读更多*。

系统内核在很多情形下会发送一些exception和signal。为了简洁起见，我们只讨论一些常见的，导致进程终止，或者crash中感兴趣的部分进行分析。

##Signals
下面是常遇到的，进程终止的signal一个清单：



<table>
   <tr>
      <td>Signal</td>
      <td>Description</td>
   </tr>
   <tr>
      <td>SIGILL</td>
      <td><div style="text-align:left">尝试去执行一个非法的（奇怪的，未知的，有特权的）指令。出现条件是，如果你的代码执行到一个无效的但可执行的内存地址。</div></td>
   </tr>
   <tr>
      <td>SIGTRAP</td>
      <td><div style="text-align:left">通常用于debugger断点和其他的debugger功能。</div></td>
   </tr>
   <tr>
      <td>SIGABRT</td>
      <td><div style="text-align:left">告诉进程终止。它只能被进程自己调用<code>abort()</code>这个C标准库函数来初始化。除非你自己使用<code>abort()</code>，大部分情况是你使用了<code>assert()</code>或者<code>NSAssert()</code>失败后产生的。</div></td>
   </tr>
   <tr>
      <td>SIGFPE</td>
      <td><div style="text-align:left">浮点或计算异常，比如除0的情况。</div></td>
   </tr>
   <tr>
      <td>SIGBUS</td>
      <td><div style="text-align:left">总线错误。比如尝试加载一个未对齐的指针。</div></td>
   </tr>
   <tr>
      <td>SIGSEGV</td>
      <td><div style="text-align:left">当内核发现进程尝试去访问无效的内存地址时，发送该信号。比如，放一个无效指针所指向的内容。</div></td>
   </tr>
</table>

一个信号有一个默认的信号处理函数或者是一个自定义的处理函数(使用`sigaction`来实现)。信号处理函数的第二个入参，是一个siginfo_t结构体，用来传入发生错误时更详细的相关信息。我们特别关注的是`si_addr`字段，它代表错误所发生的地址。下面引用了一段从内核的bsd/sys/signal.h文件中一段注释：

```
When the signal is SIGILL or SIGFPE, si_addr contains the address of the faulting
instruction. When the signal is SIGSEGV or SIGBUS, si_addr contains the address of
the faulting memory reference. Although for x86 there are cases of SIGSEGV for
which si_addr cannot be determined and is NULL.

```
翻译：


    当信号是SIGILL或者SIGFPE时，si_addr包含了错误指示的地址。当信号是SIGSEGV或者SIGBUS时，
    si_addr包含了错误内存的引用的地址。尽管对于x86来说，有很多种SIGSEGV的情况下，si_addr不能被发现并且
    传递的是NULL。


##Exceptions
On Darwin, UNIX signals are built on top of Mach Exceptions, and the kernel performs some mapping between the two. For a more comprehensive list of exception types, see osfmk/mach/exception_types.h. Again, we list only the most important exception types:

Exception	Description
EXC_BAD_ACCESS	Memory could not be accessed. The memory address where an access attempt was made is provided by the kernel.
EXC_BAD_INSTRUCTION	Instruction failed. Illegal or undefined instruction or operand.
EXC_ARITHMETIC	For arithmetic errors. The exact nature of the problem is also made available.
It is also possible for an exception to have an associated exception code that contains further information about the problem. For instance, EXC_BAD_ACCESS could point to a KERN_PROTECTION_FAILURE, which would indicate that the address being accessed is valid, but does not permit the required form of access (seeosfmk/mach/kern_return.h). EXC_ARITHMETIC exceptions will also include the precise nature of the problem as part of the exception code.

Example
The example exception section shown above is an excerpt from this crash report. We can see that the reason for the crash is a SIGABRT, which makes us think that this crash might have been caused by a failing assertion. If we inspect the exception code, we can see that the kernel included the address of the instruction in question (0x3466e32c), and the crashed thread’s index. Sure enough, if we search for that address in the report, we’ll find it in both the program counter register (see below), and the crashing thread’s stack trace:

0 libsystem_kernel.dylib 0x3466e32c ___pthread_kill + 8
In this example, we can see that there’s even more to discover in the ‘Application Specific Information’ section, which tells us that a NSInternalInconsistencyException (a Foundation exception) occurred which was not caught and led to a call to abort(), which is ultimately why we saw the SIGABRT signal.

Binary Images
At the end of a crash report, we find a list of the loaded binary images, which in essence tells us which libraries were loaded by the application, and what their address space is within the process. Each entry in this list also shows the UUID for the respective binary, which is generated and set by the linker as part of the build process. It is stored in the Mach-O binary and identified by the LC_UUID command. The UUID is the same for the binary and the .dSYM bundle generated for it, which ensures that there’s no mismatch during symbolication.

For example, we might find the following in the list of binary images:

0x35f62000 - 0x36079fff CoreFoundation armv7 /System/Library/Frameworks/CoreFoundation.framework/CoreFoundation
The first two hexadecimal numbers indicate the beginning and end of the address space that the CoreFoundation image is loaded into. If we consider a line from a stack trace such as the following, we can see that the function that was traced falls into this library’s address space:

9 CoreFoundation 0x35fee2ad ___CFRunLoopRun + 1269
When is this information useful? Imagine that for some reason we desire to analyze the assembly of one of the libraries that our application is using, let’s say CoreFoundation. CoreFoundation is not statically linked (i.e. it’s not part of the application’s own binary), but is dynamically loaded at runtime. When such loading occurs, the library’s binary image ends up at some arbitrary location in the process’ address space.

Let’s now assume that we know the value of the program counter (PC, i.e. the address of the instruction to be executed next) of the process, and that the PC refers to an instruction somewhere in CoreFoundation’s address space, relative to the process. If on our development machine we disassemble a local, on-disk copy of the CoreFoundation binary that the application previously loaded, we’d not be able to map the process-relative PC to the address space of the local copy, given that CoreFoundation was mapped to some arbitrary address. If, however, we know the offset the CoreFoundation binary image was at during the process’ lifetime, we can easily map the process-relative PC to the corresponding value for the on-disk binary.

As an example, if CoreFoundation’s binary image was loaded into our process with an offset of 0x35f62000, and the PC is 0x35fee2ad, then we can compute the actual address of the CoreFoundation instruction as:

0x35fee2ad - 0x35f62000 == 0x8c2ad
In our locally disassembled CoreFoundation binary, we can now inspect the instruction at address 0x8c2ad.

Register State
Further down in the crash report we find the ARM thread state of the crashed thread, which is essentially a list of the CPU registers and their respective values at the time of the crash. The section may look like so:

Thread 0 crashed with ARM Thread State:
r0: 0x00000000 r1: 0x00000000 r2: 0x00000001 r3: 0x00000000
r4: 0x00000006 r5: 0x3f09cd98 r6: 0x00000002 r7: 0x2fe80a70
r8: 0x00000001 r9: 0x00000000 r10: 0x0000000c r11: 0x00000001
ip: 0x00000148 sp: 0x2fe80a64 lr: 0x3526f20f pc: 0x3466e32c
cpsr: 0x00000010
A crashing thread’s register state is not always required to read a crash report, but there are certainly instances where this information can be very useful. For instance, if the crashing instruction tries to access a register that has a value of 0x0, and the thread tries to access memory at that register’s address or with only a small offset the failure cause is extremely likely to be a NULL dereference. That’s because the entire page from 0-4095 is mapped with read/write/execution permissions disabled, i.e. no access is allowed.

In the following section, we’ll give another more elaborate example.

An Example
Let us now consider a contrived example that illustrates why it can be handy to have the register state of a crashing application.

Assume that the crash report tells us the thread that the crash happened in, and that we’ve identified the line in our program that causes the crash. Imagine that the line reads as such:

new_data->ptr2 = [myObject executeSomeMethod:old_data->ptr2];
If we consider the UNIX signal that got sent (SIGSEGV), we can guess that the crash happened due to the application trying to dereference a memory address that for some reason is invalid. In this example, there are two pointers being evidently dereferenced (new_data and old_data). The question then becomes: Which one is responsible for the crash?

Assembly to the Rescue
If we still have a copy of the crashing binary, we can disassemble it and look at the exact instruction that was being executed when the application crashed (the address of which is made available as part of the SIGSEGV’ssi_addr).

Assume that the application at the time of crashing was executing the following ARM instruction:

str r0, [r1, #4]
There are two registers being used here: r0 and r1. Imagine that r1 points to the address of a C struct with the following declaration:

typedef struct { void *ptr1, void *ptr2 } data_t;
Then, given that pointers on ARM have a width of 4 bytes, we know that r1 plus four refers to the struct memberptr2.

In the form above, the str instruction takes the value stored in register r0 and attempts to store it at the address pointed to by r1, plus four. We could read the assembly like so:

*(r1 + 4) = r0;
That is, we can think of str being the equivalent of a C assignment here.

We now know two things:

The application received a SIGSEGV (invalid memory access we already knew this; see above).
The crash happened while trying to store a value to some address with an offset of 4.
At this point we should be able to suspect that the pointer value of new_data might not be what we were expecting it to be.

Looking at the ARM thread state (the registers and their respective values), we can confirm this theory:

r1: 0x00000000
In other words, we can now be certain that we are trying to dereference an address (which was computed based on an offset and a NULL pointer) which in all likelihood points to invalid memory. This was ultimately why the application crashed. If this was a real example, the next step would be to look at our code and determine what path the code could have taken such that we ended up on the crashing line with new_data == NULL, and to fix that.

Additional Notes
To drive the point home, dereferencing old_data can’t be the cause of the crash, given that the access to it only involves reading the value, not writing it (i.e. we wouldn’t be seeing a str instruction). That said, one should not be confused when looking at a more complete list of ARM assembly instructions, where one might see an instruction such as

str [r2, #4], r0
This is an example of how the argument is passed to the subroutine (assuming r2 points to a struct of typedata_t). That is, the value of the second struct member is stored in register r0 prior to jumping into the subroutine, which then performs its work.

When the subroutine is about to return (assuming the Apple ARM ABI), the return value of the function is placed in r0 (if it fits there). In the above case, that would be the value returned by the executeSomeMethod: call, which by the time the crashing instruction executes is already stored in r0.

Conclusion
A fair number of crashes are easy to comprehend. For many crashes, however, comprehensive data is required for crash analysis. In those cases, a reliable crash reporting facility is desirable. Since you can’t know in advance when a crash will occur, and because you might not be able to narrow the cause down without a report, it’s advisable to set up a crash reporting solution for your app early during development. Using iTunes Connect does get you crash reports, but you can’t use it before your app is on the App Store, which means it’s not suitable for beta testing.

Using a service such as HockeyApp has the following advantages over iTunes Connect:

You can get crash reports even during beta testing, before the app is on the iTunes Store.
You can access crash reports easily through a convenient web interface.
You can get notified via mail as soon as a crash happens.
Crash reports have already been symbolicated for you (assuming proper setup).
Users often opt-out of Apple’s data gathering, which means you wouldn’t get any crash reports. This is because Apple asks the user for permission to “improve its products and services by automatically sending daily diagnostics and usage data” once when a device is first used, which is a global setting that doesn’t easily convey the effect of declining the request.
It is therefore a good idea to find a crash reporting solution that works for you as soon as you deploy your application. (In the interest of full disclosure: HockeyApp has been a great sponsor of our open-source work on PLCrashReporter).

We hope that we have succeeded in shedding some light on the more advanced information provided by crash reports. In case you need some expert guidance regarding (our) crash reporting solutions, please note that we’re available for hire.
