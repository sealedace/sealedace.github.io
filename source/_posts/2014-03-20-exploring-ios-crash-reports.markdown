---
layout: post
title: "Exploring iOS Crash Reports"
date: 2014-03-20 11:25:11 +0800
comments: true
published: false
categories: iOS
---


[original article]: https://www.plausible.coop/blog/?p=176
[crash report]: https://gist.github.com/dennda/56851a63d935b54d3147

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
在Darwin上，UNIX信号是创建在Mach异常的最顶层，同时，内核在两者之间做了一些映射。更多的异常类型，可以查看`osfmk/mach/exception_types.h`文件。再一次，我们仅仅列出了最重要的异常类型：

<table>
   <tr>
      <td>Exception</td>
      <td>Description</td>
   </tr>
   <tr>
      <td>EXC_BAD_ACCESS</td>
      <td><div style="text-align:left">内存不可访问。尝试访问的地址是通过内核提供的。</div></td>
   </tr>
   <tr>
      <td>EXC_BAD_INSTRUCTION</td>
      <td><div style="text-align:left">指令失败。非法或者未定义的指令或者运算。</div></td>
   </tr>
   <tr>
      <td>EXC_ARITHMETIC</td>
      <td><div style="text-align:left">算法错误。</div></td>
   </tr>
</table>

当然，也有可能一个异常有一个相关的异常代码，这个异常代码包含了问题的一些信息。举一个例子，`EXC_BAD_ACCESS`可能指向一个代码叫`KERN_PROTECTION_FAILURE`，它暗示着，被访问的地址是有效的，但是权限不够，不能够访问（查看osfmk/mach/kern_return.h）。EXC_ARITHMETIC类型的异常可以包含问题的本质作为异常代码的一部分。

####Example
上面的异常示例内容来自于这份[crash report][crash report]。我们能看到crash的原因是`SIGABRT`，这让我们想到可能是断言（assert）导致的。如果我们仔细观察分析这个异常的代码，我们可以看到内核包含了问题所在的地址（0x3466e32c），以及crash的线程索引。非常肯定的是，如果我们在报告中搜索这个地址，我们会发现它同时存在于程序计数寄存器，和crash的线程堆栈信息：
	
	0 libsystem_kernel.dylib 0x3466e32c ___pthread_kill + 8
	
在这个例子中，我们看到在'Application Specific Information'部分可以找到更多的信息。这些信息告诉我们一个叫`NSInternalInconsistencyException`（一种Foundation异常）的异常出现了，但是没有被捕获到，而它是导致了程序的退出的原因，这也就是为什么我们最终看到的是SIGABRT信号。

###Binary Images
在报告的最后，我们发现了一个记录了加载二进制镜像数据的列表，它告诉了我们哪些库被加载了，以及它们在这个进程中的地址空间。这个列表中每一项也记录了标记各自二进制数据的UUID，这些UUID由链接器生成并分配，同时这个步骤也变成了build过程的一部分。它被`LC_UUID`命令分配了UUID，并且存储在了Mach-O文件中。这个UUID跟`.dSYM`生成的是一样的，这样保证了在符号化的过程中不会出现不匹配。

举个例子，我们不难找到类似下面的二进制镜像：

```
0x35f62000 - 0x36079fff CoreFoundation armv7 /System/Library/Frameworks/CoreFoundation.framework/CoreFoundation
```

前面的两个十六进制的数字表示CoreFoundation库加载内存中的起始地址和末尾地址。

如果我们假设在栈列表中有如下的信息，我们可以看到追踪到的函数落在这个库（CoreFoundation）的地址空间内：
```
9 CoreFoundation 0x35fee2ad ___CFRunLoopRun + 1269
```

这些信息什么时候对我们有用？想象下一种情况，我们急需分析其中一个应用中正在使用的库，就拿CoreFoundation来说。CoreFoundation并不是静态链接（也就是说，它不被包含在app的二进制文件里面），它是在runtime的时候被动态加载的。当出现这种动态加载的时候，库的二进制数据在进程的地址空间的位置是随机的。

现在让我们假设已知当前程序计数器的值（PC，下一个被执行的指令的地址），并且PC指向相对于进程的CoreFoundation的地址空间中某一条指令的位置。如果我们开发机器使用了一份应用程序已经加载本地磁盘上CoreFoundation库的拷贝，我们是不能把相对进程的PC映射到本地拷贝库的地址空间的，因为CoreFoundation是被加载到一个随机的地址上的。不管怎样，假如，我们知道了CoreFoundation库二进制数据在进程生命周期中的地址偏移量，我们可以容易地将相对进程的PC映射到对应的磁盘上的二进制数据。

举个例子，如果CoreFoundation的库被加载到我们的进程中的地址偏移量是0x35f62000，并且PC是0x35fee2ad，这样我们可以计算出CoreFoundation指令的实际地址为：

	0x35fee2ad - 0x35f62000 == 0x8c2ad
在本地CoreFoundation库文件中，我们现在可以查看到地址为0x8c2ad的指令了。

####Register State
进一步的深入下去，我们找到了ARM线程在crash的时候的状态。其实本质上就是一张表，上面列出了CPU寄存器以及它们在crash的时候的状态值。类似下面这段：

```
Thread 0 crashed with ARM Thread State:
r0: 0x00000000 r1: 0x00000000 r2: 0x00000001 r3: 0x00000000
r4: 0x00000006 r5: 0x3f09cd98 r6: 0x00000002 r7: 0x2fe80a70
r8: 0x00000001 r9: 0x00000000 r10: 0x0000000c r11: 0x00000001
ip: 0x00000148 sp: 0x2fe80a64 lr: 0x3526f20f pc: 0x3466e32c
cpsr: 0x00000010
```
当我们拿到crash报告阅读分析的时候，上面的信息一般是用不到的。但是，在某些特定的情况下，这些信息对我们也许是非常有用的。比如，导致crash的那条指令正在访问一个值为0x0的寄存器，并且线程正在访问那个寄存器地址（或者有非常小偏移的地址）的内存，这个将会导致类似一种对NULL取值的情况。因为整个页面从0-4095都是被映射为非读/写/执行的权限，也就是说是禁止任何访问的。

在下面的段落中，我们将给出一个更详细的例子。

#####An Example
我们现在来考虑一种情况，为什么在应用崩溃的时候知道寄存器的状态会对我们有帮助。

假设crash report告诉了我们程序crash的线程，并且我们已经定位到了导致crash的那行代码。假设这行代码如下：

	new_data->ptr2 = [myObject executeSomeMethod:old_data->ptr2];
如果发送出的UNIX信号为`SIGSEGV`，我们可以猜到程序崩溃的原因是，程序在对一个无效的内存地址进行取值操作。很明显地，在这个例子中有两个指针在被执行取值操作（new_data和old_data）。那么，现在的问题变成了：是哪一个导致了crash？

######Assembly to the Rescue
如果我们手头上仍然保留了一份崩溃的程序的拷贝，我们可以分解它，然后找到那个在程序崩溃的时候正在执行的指令（也就是SIGSEGV的si_addr的部分可用的地址）。

假设程序在崩溃的时候执行如下的ARM指令：

	str r0, [r1, #4]
这里有2个寄存器正在被使用：`r0`和`r1`。想象r1指向的地址是一个如下定义的C结构体：

	typedef struct { void *ptr1, void *ptr2 } data_t;
接着，鉴于ARM上的指针长度位是4字节，我们知道r1加上4指向了结构体成员`ptr2`。
在的代码中，`str`这个指令将r0中的值存储到r1中的地址加上4的地址里。我们可以转换一下，如下：

	*(r1 + 4) = r0;
也就是说，我们可以把`str`理解等价成C格式的赋值。

现在，我们明白了两件事情：

* 程序接收到**SIGSEGV**信号（无效地址访问）
* 当在尝试存储一个值到某个地址偏移4的地方的时候，程序出现了crash

这个时候，我们应该可以猜测到`new_data`的指针的值可能并不是我们所期望的那个值。
从这个ARM线程的状态中（即，寄存器和它们的值），我们可以确定这样的理论：

	r1: 0x00000000
换句话说，我们现在可以确定的是我们正在尝试对一个各种可能性指向无效内存的地址（根据对NULL指针和偏移量计算得来的地址）进行取值操作。这个就是导致应用crash的根本原因。如果这个例子放在现实中的话，那么我们下一步要做的就是查看我们的代码，并找到让代码执行到`new_data == NULL`这行操作的路径，然后解决问题。

######Additional Notes
为了把问题分析到位，其实对`old_data`进行取值并不会导致crash，因为这种访问只做读取，没有写入（换句话说，我们不会遇到str指令）。
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
