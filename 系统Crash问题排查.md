---
title: 系统Crash问题排查
date: 2018-09-26 09:30:08
tags: JVM
---

翻译：曲风富

_译者注：本文翻译自Oracle官方文档，有一定英文能力者建议查看原文：_
[https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/crashes.html](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/crashes.html)

本章节提供针对系统crash诊断的某些特定过程的信息和指南。

崩溃或致命错误会导致进程异常退出。导致crash的原因有很多，比如HotSpot VM、系统库、Java SE库或API、应用本地代码甚至操作系统中的某个bug，都有可能会导致crash。外部因素，诸如系统资源耗尽也可能会导致crash。

由HotSpot VM或Java SE库中的代码中的bug引发的        比较少见。因此本章节将针对如何检查crash提供一些建议。在某些情况下，围绕一个crash现象努力直到bug被诊断出来并修复是有可能的。

通常分析crash的第一步是定位到致命错误的日志。这个日志是一个文本文件，是HotSpot VM在crash时生成的。如何定位到这个文件以及此日志的详细描述，请参见这篇文章：[Appendix C, Fatal Error Log](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/felog.html)。

## 1 crash采样 ##

本节提供了一些演示如何使用error log去找到crash原因的例子。

### 1.1 确定crash发生的位置 ###

Crash日志的头部指出了有问题的帧。

如果帧类型是本地帧并且不是操作系统本地帧，那么说明问题可能出自本地库并且不是Java虚拟机导致。解决这种crash的第一步是查看crash发生处的本地帧的代码。根据本地库的代码，这里有3个选项：

1. 如果本地库由你的应用提供，那么请检查你的本地库的代码。-Xcheck:jni可以帮你很多本地bug。详情请见：[2.1 ](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/clopts.html#gbmtq)[-Xcheck:jni](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/clopts.html#gbmtq)[ Option](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/clopts.html#gbmtq)；
2. 如果你应用中使用的本地库由第三方提供，建议向第三方提供bug report，并提供致命错误日志；
3. 如果本地库是JRE的一部分，那么请向Java社区提交bug report，并确保库的名称明确无误，以便bug report被分派给正确的开发人员。

如果错误日志中的顶部帧信息显示是其他类型的帧，请向Java社区提交bug report、错误日志以及如何复现问题的相关信息。

另外，请参阅本章剩余的内容。

## 1.2 本地代码中Crash ##

如果致命错误日志显示crash发生在本地库，那么很有可能是本地代码或者JNI库代码中的bug导致。Crash当然也可能是其他的原因导致，但是对库、core文件、crash导出文件的分析是一个好的开端。例如，思考一下下面的从致命错误日志头部提取出来的信息：

```
# An unexpected error has been detected by HotSpot Virtual Machine:
#
#  SIGSEGV (0xb) at pc=0x417789d7, pid=21139, tid=1024
#
# Java VM: Java HotSpot(TM) Server VM (6-beta2-b63 mixed mode)
# Problematic frame:
# C  [libApplication.so+0x9d7]
```

在这个例子中，一个在libApplication.so库中执行的线程发生了SIGSEGV错误。

在某些场景中某个本地库中的bug显示在Java虚拟机中crash。请看下面的crash：一个Java线程在_thread_in_vm状态下失败(即它在Java VM代码中执行)

```
# An unexpected error has been detected by HotSpot Virtual Machine:
#
#  EXCEPTION_ACCESS_VIOLATION (0xc0000005) at pc=0x08083d77, pid=3700, tid=2896
#
# Java VM: Java HotSpot(TM) Client VM (1.5-internal mixed mode)
# Problematic frame:
# V  [jvm.dll+0x83d77]

---------------  T H R E A D  ---------------

Current thread (0x00036960):  JavaThread &quot;main&quot; [_thread_in_vm, id=2896]
 :
Stack: [0x00040000,0x00080000),  sp=0x0007f9f8,  free space=254k
Native frames: (J=compiled Java code, j=interpreted, Vv=VM code, C=native code)
V  [jvm.dll+0x83d77]
C  [App.dll+0x1047]          <========= C/native frame
j  Test.foo()V+0
j  Test.main([Ljava/lang/String;)V+0
v  ~StubRoutines::call_stub
V  [jvm.dll+0x80f13]
V  [jvm.dll+0xd3842]
V  [jvm.dll+0x80de4]
V  [jvm.dll+0x87cd2]
C  [java.exe+0x14c0]
C  [java.exe+0x64cd]
C  [kernel32.dll+0x214c7]
 :
```

在这个case中，栈跟踪显示一个在App.dll中的本地程序调用了VM(可能通过JNI)。

如果你遇到了一个在本地库中的crash(如上面所述的例子)，如果可能的话，你可以使用本地的调试程序链接到core文件或者crash导出文件。本地调试程序有dbx、gdb或windbg，因操作系统而异。

另一个方法是在启动时将-Xcheck:jni添加到启动命令行中(参见[B.2.1 ](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/clopts.html#gbmtq)[-Xcheck:jni](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/clopts.html#gbmtq)[ Option](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/clopts.html#gbmtq))。这个选项不保证发现所有的JNI代码中的问题，但它可以帮你定位到很多的问题。

如果crash的本地库是Java运行环境的一部分(如awt.dll, net.dll等)，那么很可能你遇到了一个Java库或API的bug。如果经过进一步分析之后你断定这是个Java库或API的bug，请收集尽可能多的数据并提交bug或寻求Java技术支持。详情参见[Chapter 7, Submitting Bug Reports](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/bugreports.html)。

### 1.3 栈溢出导致Crash ###

Java语言代码中的栈溢出(也叫爆栈)会导致线程抛出java.lang.StackOverflowError。换句话说，C和C++写入时达到了栈的末尾并且引发了栈溢出。这个致命错误导致了进程终止。

在HotSpot实现中，Java方法与C/C++本地代码共享栈帧，也就是用户本地代码和虚拟机本身。Java方法生成代码来检查栈空间有可用的到栈尾的固定距离，来确保本地代码在不超出栈空间的情况下能够被执行。这个到栈尾的距离被称为"Shadow Pages"。这个距离是可调节的，这样应用需要比默认值更大的距离时就可以增加Shadow page的尺寸。增大Shadow pages的选项是-XX:StackShadowpages=_n_，n的值要大于shadow pages在当前平台下的默认值。

如果你的应用出现了分段错误，而且没有生成core文件或error日志(见[Appendix C, Fatal Error Log](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/felog.html))或在Windows系统中发现STACK_OVERFLOW_ERROR或"An irrecoverable stack overflow has occurred"消息，这说明StackShadowPages已经耗尽需要扩容。

如果你想要增大StackShadowPages 的值，你也需要使用-Xss参数增加默认线程栈大小。增加线程栈大小可能会导致能够创建的线程数的减少。默认线程栈大小根据平台差异从256K到1024K不等。

下面的代码片段来自于Windows系统的一个致命错误日志，其中的一个线程引发了本地代码中的栈溢出。
```
# An unexpected error has been detected by HotSpot Virtual Machine:
#
#  EXCEPTION_STACK_OVERFLOW (0xc00000fd) at pc=0x10001011, pid=296, tid=2940
#
# Java VM: Java HotSpot(TM) Client VM (1.6-internal mixed mode, sharing)
# Problematic frame:
# C  [App.dll+0x1011]
#

---------------  T H R E A D  ---------------

Current thread (0x000367c0):  JavaThread &quot;main&quot; [_thread_in_native, id=2940]
:
Stack: [0x00040000,0x00080000),  sp=0x00041000,  free space=4k
Native frames: (J=compiled Java code, j=interpreted, Vv=VM code, C=native code)
C  [App.dll+0x1011]
C  [App.dll+0x1020]
C  [App.dll+0x1020]
:
C  [App.dll+0x1020]
C  [App.dll+0x1020]
...<more frames>...

Java frames: (J=compiled Java code, j=interpreted, Vv=VM code)
j  Test.foo()V+0
j  Test.main([Ljava/lang/String;)V+0
v  ~StubRoutines::call_stub
```

下面是上面的日志包含的信息：

- 异常是EXCEPTION_STACK_OVERFLOW；
- 线程的状态是_thread_in_native，说明线程正在执行本地或者JNI代码；
- 栈信息显示，栈的可用空间是4K(Windows中单页的大小)。另外，栈指针(Stack pointer, sp)在0x00041000的位置，已经很接近栈尾了(0x00040000)(译者注: java线程栈是从高地址往低地址方向走的)；
- 本地帧的输出显示一个递归的本地函数是这个case的问题；
- 标注…<more frames>…表明还有更多的帧存在但没有被输出。最多可输出100帧。

_译者注：关于栈溢出的问题，感兴趣的同学可以进一步阅读这篇文章：_[https://www.jianshu.com/p/debef4f69a90](https://www.jianshu.com/p/debef4f69a90)。

### 1.4 HotSpot编译线程中Crash ###

如果错误日志中显示crash时当前Java线程的名称是CompilerThread0, CompilerThread1, 或AdapterCompiler，那么可能你遇到了一个编译bug。这种情况下你有必要临时性地切换编译器(例如把HotSpot Server VM替换成HotSpot Client VM，反之亦然)，或者将引发crash的方法从编译过程中排除。详情见：[4.2.1 Crash in HotSpot Compiler Thread or Compiled Code](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/crashes.html#gbyzd)。

### 1.5 编译的代码中Crash ###

如果编译的代码中发生crash，那么你可能遇到了编译器bug导致了错误的代码生成的问题。你可以识别出一个在被编译的代码中出现的crash，如果有问题的帧被标注为代码J(代表被编译的Java帧)。下面是一个这样的crash例子：
```
# An unexpected error has been detected by HotSpot Virtual Machine:
#
#  SIGSEGV (0xb) at pc=0x0000002a99eb0c10, pid=6106, tid=278546
#
# Java VM: Java HotSpot(TM) 64-Bit Server VM (1.6.0-beta-b51 mixed mode)
# Problematic frame:
# J  org.foobar.Scanner.body()V
#
:
Stack: [0x0000002aea560000,0x0000002aea660000),  sp=0x0000002aea65ddf0,
  free space=1015k
Native frames: (J=compiled Java code, j=interpreted, Vv=VM code, C=native code)
J  org.foobar.Scanner.body()V

[error occurred during error reporting, step 120, id 0xb]
```

需要注意的是一个完整的线程栈无法提供。输出行"error occurred during error reporting"表示在尝试获得栈跟踪的时候出错了(本示例中可能是栈损坏)。

这种情况下你有必要临时性地切换编译器(例如把HotSpot Server VM替换成HotSpot Client VM，反之亦然)，或者将引发crash的方法从编译过程中排除。在本示例中，不太可能将编译器将64位Server VM切换，因为将其切换至32位Client VM不太可行。

### 1.6 VMThread中Crash ###

如果log输出显示当前线程是VMThread，那么请在日志中的THREAD区域寻找包含VM_Operation的行。VMThread是HotSpot虚拟机中一个特殊的线程。它向虚拟机中提交诸如GC这种特殊的任务。如果VM_Operation显示其操作是垃圾回收，那么很有可能你遇到了诸如堆损坏的问题。

Crash也有可能有GC问题引起，但这可以等价于一些其他问题(比如编译器或运行期bug)导致堆中的对象引用处于不连续或者不正确的状态。在这个场景中，尽可能多地收集与环境相关的信息，并尝试可能的替代方案。如果这个问题是GC相关，你可能需要临时性地修改GC配置作为替代方案。这方面的内容在[4.2.2 Crash During Garbage Collection](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/crashes.html#gbyzq)中有讨论。

## 2 寻求替代方案 ##

如果是重要的应用发生了crash，并且是由HotSpot VM中的bug引发，那么你应该快速寻找一个替代方案。本节的目标是给出一些可能的替代方案。如果crash的应用是部署在JDK最近的发布版本上，那么这个crash事件应该报告给Oracle。

如果关键应用程序发生崩溃，并且崩溃似乎是由HotSpot VM中的错误引起的，那么可能需要快速找到临时解决方法。 本节的目的是提出一些可能的解决方法。 如果使用最新版本的JDK部署的应用程序发生崩溃，则应始终向Oracle报告崩溃

_注意 - 即使本节中的相关内容成功消除了崩溃，但是问题的解决方案不是固定的，而是临时解决方案。 寻求电话支持或提交包含了能说明问题的原始配置的BUG报告。_

### 2.1 在HotSpot编译线程或编译代码中crash ###

如果致命错误日志显示crash发生在编译器线程中, 则可能是遇到了编译器 bug (但并非总是如此)。同样, 如果在编译后的代码中crash, 则可能是因为编译器生成了不正确的代码所致。

在HotSpot VM (-client选项) 的情况下, 编译器线程以 CompilerThread0 的形式出现在错误日志中。在HotSpot Server VM中, 则有多个编译器线程, 它们在错误日志文件中显示为 CompilerThread0、CompilerThread1 和 AdapterThread。

下面是 J2SE 5.0 开发过程中遇到和修复的编译器bug的错误日志片段。日志文件显示使用了HotSpot Server VM, 并且崩溃发生在 CompilerThread1 中。此外, 日志文件显示当前 CompileTask 是 java.lang.Thread.setPriority方法的编译。
```
# An unexpected error has been detected by HotSpot Virtual Machine:
#
:
# Java VM: Java HotSpot(TM) Server VM (1.5-internal-debug mixed mode)
:
---------------  T H R E A D  ---------------

Current thread (0x001e9350): JavaThread "CompilerThread1" daemon [_thread_in_vm, id=20]

Stack: [0xb2500000,0xb2580000),  sp=0xb257e500,  free space=505k

Native frames: (J=compiled Java code, j=interpreted, Vv=VM code, C=native code)

V  [libjvm.so+0xc3b13c]
:

Current CompileTask:
opto: 11      java.lang.Thread.setPriority(I)V (53 bytes)

---------------  P R O C E S S  ---------------

Java Threads: ( => current thread )
  0x00229930 JavaThread "Low Memory Detector" daemon [_thread_blocked, id=21]
=>0x001e9350 JavaThread "CompilerThread1" daemon [_thread_in_vm, id=20]
 :
```
在这种情况下, 有两种潜在的变通方法：

- 蛮力方法: 更改jvm配置, 以便使用-client选项运行应用程序以指定HotSpot Client VM;
- 假定bug仅在setPriority方法的编译过程中发生, 并将此方法从编译中排除。

在某些环境中配置第一种方法（使用-client选项）可能很简单。在其他情况下，如果配置复杂或者无法轻松访问配置VM的命令行，则可能会更加困难。通常，从HotSpot Server VM切换到HotSpot Client VM也会降低应用程序的峰值性能。 根据环境的不同，在诊断和修复实际问题之前，这可能是可以接受的。

第二种方法（从编译中排除方法）需要在应用程序的工作目录中创建文件.hotspot_compiler。 以下是此文件的示例
```
exclude    java/lang/Thread    setPriority
```
通常，此文件的格式为exclude CLASS METHOD，其中CLASS是类（使用包名称完全限定），METHOD是方法的名称。构造方法指定为<init>，静态初始化程序指定为<clinit>。

_注意 - .HOTSPOT_COMPILER文件是一个不受支持的接口。 此处仅为了故障排除和查找临时解决方案而对其进行了文档记录。_

重新启动应用程序后，编译器将不会尝试编译.hotspot_compiler文件中列出的任何排除方法。在某些情况下，这可以提供临时缓解，直到诊断出崩溃的根本原因并修复错误

为了验证HotSpot VM是否正确定位并处理了上述示例中显示的.hotspot_compiler文件，请在运行时查找以下日志信息。请注意，文件名分隔符是一个点，而不是斜杠。
```
### Excluding compile:    java.lang.Thread::setPriority
```
### 2.2 GC时crash ###

如果在垃圾回收（GC）期间发生崩溃，则致命错误日志会报告VM_Operation正在进行中。出于本讨论的目的，假设大多数并发GC（-XX：+ UseConcMarkSweep）未使用。VM_Operation显示在日志的THREAD部分中，表示以下情况之一：

- 用于分配的生成集合
- 全集代收藏
- 并行gc分配失败
- 并行gc未能永久分配
- 并行gc系统gc

很可能日志中报告的当前线程是VMThread。 这是用于在HotSpot VM中执行特殊任务的特殊线程。 致命错误日志的以下片段显示了串行垃圾收集器中崩溃的示例
```
---------------  T H R E A D  ---------------

Current thread (0x002cb720):  VMThread [id=3252]

siginfo: ExceptionCode=0xc0000005, reading address 0x00000000

Registers:
EAX=0x0000000a, EBX=0x00000001, ECX=0x00289530, EDX=0x00000000
ESP=0x02aefc2c, EBP=0x02aefc44, ESI=0x00289530, EDI=0x00289530
EIP=0x0806d17a, EFLAGS=0x00010246

Top of Stack: (sp=0x02aefc2c)
0x02aefc2c:   00289530 081641e8 00000001 0806e4b8
0x02aefc3c:   00000001 00000000 02aefc9c 0806e4c5
0x02aefc4c:   081641e8 081641c8 00000001 00289530
0x02aefc5c:   00000000 00000000 00000001 00000001
0x02aefc6c:   00000000 00000000 00000000 08072a9e
0x02aefc7c:   00000000 00000000 00000000 00035378
0x02aefc8c:   00035378 00280d88 00280d88 147fee00
0x02aefc9c:   02aefce8 0806e0f5 00000001 00289530
Instructions: (pc=0x0806d17a)
0x0806d16a:   15 08 83 3d c0 be 15 08 05 53 56 57 8b f1 75 0f
0x0806d17a:   0f be 05 00 00 00 00 83 c0 05 a3 c0 be 15 08 8b

Stack: [0x02ab0000,0x02af0000),  sp=0x02aefc2c,  free space=255k

Native frames: (J=compiled Java code, j=interpreted, Vv=VM code, C=native code)
V  [jvm.dll+0x6d17a]
V  [jvm.dll+0x6e4c5]
V  [jvm.dll+0x6e0f5]
V  [jvm.dll+0x71771]
V  [jvm.dll+0xfd1d3]
V  [jvm.dll+0x6cd99]
V  [jvm.dll+0x504bf]
V  [jvm.dll+0x6cf4b]
V  [jvm.dll+0x1175d5]
V  [jvm.dll+0x1170a0]
V  [jvm.dll+0x11728f]
V  [jvm.dll+0x116fd5]
C  [MSVCRT.dll+0x27fb8]
C  [kernel32.dll+0x1d33b]

VM_Operation (0x0373f71c): generation collection for allocation, mode:

 safepoint, requested by thread 0x02db7108
```
_注意 - 垃圾收集期间的crash并不意味着这是垃圾收集过程中的一个BUG。它还可能指向编译器或运行时错误或其他一些问题。_

如果在垃圾回收期间遇到重复崩溃，您可以尝试以下变通方法：

- 切换GC配置。 例如，如果您使用的是串行收集器，请尝试使用吞吐量收集器，反之亦然；
- 如果您使用的是HotSpot Server VM，请尝试使用HotSpot Client VM。

如果您不确定正在使用哪个垃圾收集器，则可以使用Solaris OS和Linux上的jmap实用程序（请参阅[2.7 ](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/tooldescr.html#gbdid)[jmap](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/tooldescr.html#gbdid)[ Utility](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/tooldescr.html#gbdid)）从核心文件获取堆信息（如果核心文件可用）。 通常，如果未在命令行中指定GC配置，则将在Windows上使用串行收集器。 在Solaris OS和Linux上，它取决于计算机配置。 如果机器至少有2GB内存且至少有2个处理器，则将使用吞吐量收集器（并行GC）。 对于较小的机器，串行收集器是默认的。选择串行收集器的选项是-XX:+UseSerialGC和选择吞吐量收集器的选项是-XX:+UseParallelGC。 如果作为解决方法，您从吞吐量收集器切换到串行收集器，那么您可能会在多处理器系统上遇到性能下降。 在诊断和解决根问题之前，这可能是可以接受的。

## 2.3 Class数据共享时crash

类数据共享是J2SE 5.0中的一项新功能。 使用Sun提供的安装程序在32位平台上安装JRE时，安装程序会将一组类从系统JAR文件加载到专用内部表示形式，并将该表示形式转储到名为共享存档的文件中。 启动VM时，共享存档将进行内存映射。这样可以节省类加载，并允许在多个VM实例之间共享与类关联的大部分元数据。 在J2SE 5.0中，仅在使用HotSpot Client VM时才启用类数据共享。 此外，只有串行垃圾收集器才支持共享。

致命错误日志在日志标题中打印版本字符串。 如果启用了共享，则由文本共享指示，如以下示例所示：
```
# An unexpected error has been detected by HotSpot Virtual Machine:
#
#  EXCEPTION_ACCESS_VIOLATION (0xc0000005) at pc=0x08083d77, pid=3572, tid=784
#
# Java VM: Java HotSpot(TM) Client VM (1.5-internal mixed mode, sharing)
# Problematic frame:
# V  [jvm.dll+0x83d77]
```
可以通过在命令行上提供-Xshare:off选项来禁用共享。 如果在禁用共享的情况下无法复制崩溃但可以在启用共享的情况下复制崩溃，则可能是您遇到此功能中的错误。在这种情况下，尽可能多地收集信息并提交错误报告。

## 3 微软C++版本的考虑

JDK 7软件是在Windows上使用Microsoft Visual Studio 2010专业版为32位和64位平台构建的。如果遇到Java 应用程序崩溃, 并且你有使用不同版本的编译器编译的本地或JNI库, 则必须考虑运行时之间的兼容性问题。具体地说, 只有在处理多个运行时时遵循Microsoft准则时, 才会支持你的环境。例如, 如果你使用一个运行时分配内存, 则必须使用同一运行时释放它。如果使用不同的库释放资源, 而不是分配资源时所使用的库, 则可能会出现不可预知的行为或崩溃。
