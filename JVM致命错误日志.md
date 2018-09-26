---
title: JVM致命错误日志
date: 2018-09-26 10:29:58
tags:
---

翻译:曲风富

_译者注：本文翻译自Oracle官方文档，有一定英文能力者建议查看原文：_
[https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/felog.html](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/felog.html)

当JVM出现致命错误时，会生成一个包含有错误信息以及出错时系统状态的日志文件。

本附录包含以下内容：

- [1 致命错误日志的位置](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/felog.html#gbwcy)
- [2 致命错误日志描述](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/felog.html#gbmuz)
- [3 头部格式](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/felog.html#header)
- [4 线程部分格式](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/felog.html#gbvki)
- [5 进程部分格式](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/felog.html#gbvkd)
- [6 系统部分格式](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/felog.html#gbvla)

## C.1 致命错误日志的位置

产品标志-XX:ErrorFile=_file_ 可用于指定文件的创建位置，其中_file_表示文件位置的完整路径。文件变量中的子字符串%%将转换为%，子字符串%p将转换为进程的进程ID。

在下面的示例中，错误日志文件将写入目录/var/log/java，并将命名为java_error{pid}.log。
```
java -XX:ErrorFile=/var/log/java/java_error%p.log
```
如果未指定-XX:ErrorFile=file标志，则默认情况下文件名为hs_err_{pid}.log，其中pid是进程的进程ID。

此外，如果未指定-XX:ErrorFile=file标志，系统将尝试在进程的工作目录中创建该文件。如果无法在工作目录中创建文件（空间不足，权限问题或其他问题），则会在操作系统的临时目录中创建该文件。在Solaris OS和Linux上，临时目录是/tmp。 在Windows上，临时目录由TMP环境变量的值指定; 如果未定义该环境变量，则使用TEMP环境变量的值。

## C.2 致命错误日志描述

错误日志包含致命错误发生时获取的信息，包括以下可能的信息：

- 引发致命错误的操作异常或信号
- 版本和配置信息
- 引发致命错误和线程堆栈跟踪的线程的详细信息
- 正在运行的线程列表及其状态
- 有关堆的摘要信息
- 已加载的本地库列表
- 命令行参数
- 环境变量
- 有关操作系统和CPU的详细信息

_注 - 在某些情况下，只有这些信息的子集输出到错误日志。这可能会发生在致命错误非常严重，以至于错误的处理程序无法恢复并报告所有细节。_

错误日志是一个文本文件，包含以下部分：

- 标题，提供crash的简要说明。[3 Header Format](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/felog.html#header)
- 包含线程信息的部分。[4 Thread Section Format](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/felog.html#gbvki)
- 包含进程信息的部分。[5 Process Section Format](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/felog.html#gbvkd)
- 包含系统信息的部分。[6 System Section Format](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/felog.html#gbvla)

_注 - 请注意，此处描述的致命错误日志的格式基于JDK7格式，可能与其他版本有所不同。_

## C.3 头部格式

每个致命错误日志文件开头的头部区域都包含问题的简要说明。这部分头部信息也打印到标准输出，可能会显示在应用程序的输出日志中。

头部信息包含指向HotSpot虚拟机错误报告页面的链接，用户可以在其中提交错误报告。

以下是crash日志的示例头部信息：
```
#
# An unexpected error has been detected by Java Runtime Environment:
#
#  SIGSEGV (0xb) at pc=0x417789d7, pid=21139, tid=1024
#
# Java VM: Java HotSpot(TM) Client VM (1.6.0-rc-b63 mixed mode, sharing)
# Problematic frame:
# C  [libNativeSEGV.so+0x9d7]
#
# If you would like to submit a bug report, please visit:
#   http://java.sun.com/webapps/bugreport/crash.jsp
#
```
此示例显示VM在一个意外信号上crash。下一行描述了信号的类型，产生信号的程序计数器（pc），进程ID和线程ID，如下所示：
```
#  SIGSEGV (0xb) at pc=0x417789d7, pid=21139, tid=1024
      |      |           |             |         +--- thread id
      |      |           |             +------------- process id
      |      |           +--------------------------- program counter
      |      |                                        (instruction pointer)
      |      +--------------------------------------- signal number
      +---------------------------------------------- signal name
```
下一行包含VM版本（Client VM或Server VM），指示应用程序是以混合模式还是解释模式运行，以及是否启用了类文件共享的指示。
```
# Java VM: Java HotSpot(TM) Client VM (1.6.0-rc-b63 mixed mode, sharing)
```
下一个信息是导致crash的函数帧，如下所示。
```
# Problematic frame:
# C  [libNativeSEGV.so+0x9d7]
  |              +-- Same as pc, but represented as library name and offset.
  |                  For position-independent libraries (JVM and most shared
  |                  libraries), it is possible to inspect the instructions
  |                  that caused the crash without a debugger or core file
  |                  by using a disassembler to dump instructions near the
  |                  offset.
  +----------------- Frame type
```
在该示例中，C帧类型指示本机C帧。下表显示了可能的帧类型。

| **帧类型** | **描述** |
| --- | --- |
| C | 本地C帧 |
| j | 解释执行的Java帧 |
| V | 虚拟机帧 |
| v | VM生成的stub帧 |
| J | 其他的帧类型，包含被编译过的Java帧 |

内部错误将导致VM错误处理程序生成类似的错误转储。但是头部格式不尽相同。 示例中的内部错误是guarantee()失败，断言失败，ShouldNotReachHere()等。 以下是如何在头部信息中查找内部错误的示例。
```
#
# An unexpected error has been detected by HotSpot Virtual Machine:
#
# Internal Error (4F533F4C494E55583F491418160E43505000F5), pid=10226, tid=16384
#
# Java VM: Java HotSpot(TM) Client VM (1.6.0-rc-b63 mixed mode)
```
在上面的头部信息中，没有信号名称或信号编号。相反，第二行现在包含文本"内部错误"和长十六进制字符串。 此十六进制字符串对检测到错误的源模块和行号进行编码。 通常，此"错误字符串"仅对在HotSpot虚拟机上工作的工程师有用。

错误字符串对行号进行编码，因此随着每次代码更改和释放而更改。一个版本中某一个带有给定错误字符串的crash（例如1.6.0）可能与更新版本中的相同crash（例如1.6.0_01）不对应，即使他们的错误字符串相匹配。

_注意 - 不要认为在一个与错误字符串相关的情况下工作的解决方案或解决方案将在与相同错误字符串相关的另一个情况下工作。 注意以下事实：_

_•具有相同根本原因的错误可能具有不同的错误字符串。_

_•具有相同错误字符的错误可能具有完全不同的根本原因。_

_因此，在排除错误时，不应将错误字符串用作唯一的标准。_

## C.4 线程区域格式

本节包含有关刚刚crash的线程的信息。如果多个线程同时crash，则只打印一个线程。

### C.4.1 线程信息

线程部分的第一部分显示了引发致命错误的线程，如下所示。
```
Current thread (0x0805ac88):  JavaThread "main" [_thread_in_native, id=21139]
                    |             |         |            |          +-- ID
                    |             |         |            +------------- state
                    |             |         +-------------------------- name
                    |             +------------------------------------ type
                    +-------------------------------------------------- pointer
```
线程指针是指向Java VM内部线程结构的指针，这通常没什么意义，除非你正在调试实时Java VM或核心文件。

以下列表显示了可能的线程类型。

- JavaThread
- VMThread
- CompilerThread
- GCTaskThread
- WatcherThread
- ConcurrentMarkSweepThread

下面的表格描述了重要的线程状态：

| **线程状态** | **描述** |
| --- | --- |
| \_thread\_uninitialized | 线程未创建。仅在内存损坏的情况下才会发生这种情况 |
| \_thread\_new | 已创建线程但尚未启动 |
| \_thread\_in\_native | 线程正在运行本机代码。该错误可能是本机代码中的错误 |
| \_thread\_in\_vm | 线程正在运行VM代码 |
| \_thread\_in\_Java | 线程正在运行解释或编译的Java代码 |
| \_thread\_blocked | 线程被block |
| ...\_trans | 如果上述任何状态后跟字符串\_trans，则表示该线程正在更改为其他状态 |


输出中的线程ID是本地线程标识符。

如果Java线程是守护线程，则在线程状态之前打印字符串守护程序。

### C.4.2 信号信息

错误日志中的下一个信息描述了导致VM终止的意外信号。在Windows系统中，输出显示如下。
```
siginfo: ExceptionCode=0xc0000005, reading address 0xd8ffecf1
```
在上面的示例中，异常代码为0xc0000005（ACCESS_VIOLATION），当线程尝试读取地址0xd8ffecf1时发生异常。

在Solaris OS和Linux系统中，信号编号（si_signo）和信号代码（si_code）用于标识异常，如下所示。
```
siginfo:si_signo=11, si_errno=0, si_code=1, si_addr=0x00004321
```
### C.4.3 寄存器上下文

错误日志中的下一个信息显示致命错误时的寄存器上下文。此输出的确切格式取决于处理器。以下示例显示了Intel（IA32）处理器的输出。
```
Registers:
EAX=0x00004321, EBX=0x41779dc0, ECX=0x080b8d28, EDX=0x00000000
ESP=0xbfffc1e0, EBP=0xbfffc1f8, ESI=0x4a6b9278, EDI=0x0805ac88
EIP=0x417789d7, CR2=0x00004321, EFLAGS=0x00010216
```
与日志中的指令信息结合使用时，寄存器值可能很有用，如下所述。

### C.4.4 机器指令

在寄存器值之后，错误日志包含堆栈顶部，随后是系统crash时程序计数器（PC）附近的32字节指令（操作码）。可以使用反汇编程序对这些操作码进行解码，以生成crash位置周围的指令。请注意，IA32和AMD64指令的长度可变，因此在crashPC之前无法始终可靠地解码指令。

_译者注：可以使用_[_https://onlinedisassembler.com/odaweb/_](https://onlinedisassembler.com/odaweb/)_这种在线工具对指令进行反汇编，当然你也可以使用udis86(_[_http://udis86.sourceforge.net_](http://udis86.sourceforge.net/)_)，前提是你需要安装gcc自行编译才能使用。_
```
Top of Stack: (sp=0xbfffc1e0)
0xbfffc1e0:   00000000 00000000 0818d068 00000000
0xbfffc1f0:   00000044 4a6b9278 bfffd208 41778a10
0xbfffc200:   00004321 00000000 00000cd8 0818d328
0xbfffc210:   00000000 00000000 00000004 00000003
0xbfffc220:   00000000 4000c78c 00000004 00000000
0xbfffc230:   00000000 00000000 00180003 00000000
0xbfffc240:   42010322 417786ec 00000000 00000000
0xbfffc250:   4177864c 40045250 400131e8 00000000
Instructions: (pc=0x417789d7)
0x417789c7:   ec 14 e8 72 ff ff ff 81 c3 f2 13 00 00 8b 45 08
0x417789d7:   0f b6 00 88 45 fb 8d 83 6f ee ff ff 89 04 24 e8
```
### C.4.5 线程栈

在可能的情况下，错误日志中的下一个输出是线程堆栈。这包括基址和堆栈顶部的地址，当前堆栈指针以及线程可用的未使用堆栈的数量。在可能的情况下，遵循堆栈帧，最多打印100帧。 对于C/C++帧，也可以打印库名称。 重要的是要注意，在某些致命错误条件下，堆栈可能已损坏，在这种情况下，此详细信息可能不可用。
```
Stack: [0x00040000,0x00080000),  sp=0x0007f9f8,  free space=254k
Native frames: (J=compiled Java code, j=interpreted, Vv=VM code, C=native code)
V  [jvm.dll+0x83d77]
C  [App.dll+0x1047]
j  Test.foo()V+0
j  Test.main([Ljava/lang/String;)V+0
v  ~StubRoutines::call_stub
V  [jvm.dll+0x80f13]
V  [jvm.dll+0xd3842]
V  [jvm.dll+0x80de4]
C  [java.exe+0x14c0]
C  [java.exe+0x64cd]
C  [kernel32.dll+0x214c7]
Java frames: (J=compiled Java code, j=interpreted, Vv=VM code)
j  Test.foo()V+0
j  Test.main([Ljava/lang/String;)V+0
v  ~StubRoutines::call_stub
```
日志包含了2个线程栈：

- 第一个线程堆栈是Native帧，它打印显示所有函数调用的本地线程。但是，此线程堆栈不考虑运行时编译器内联的Java方法; 如果方法是内联的，它们似乎是父级堆栈框架的一部分。

本地帧的线程堆栈中的信息提供有关crash原因的重要信息。通过从上到下分析列表中的库，通常可以确定哪个库可能导致问题并将其报告给负责该库的相应组织。

- 第二个线程栈是Java帧，它打印包含内联方法的Java帧，跳过本地帧。根据crash情况，可能无法打印本地线程栈，但可能会打印Java帧。

### C.4.6 进一步的细节

如果错误发生在VM线程或编译器线程中，则可能会打印更多详细信息。例如，在VM线程的情况下，如果VM线程在致命错误时执行VM操作，则打印VM操作。 在下面的输出示例中，编译器线程引发了致命错误。 该任务是编译器任务，HotSpot Client VM正在编译方法hs101t004Thread.ackermann。

Current CompileTask:

HotSpot Client Compiler:754   b

nsk.jvmti.scenarios.hotswap.HS101.hs101t004Thread.ackermann(IJ)J (42 bytes)

对于HotSpot Server VM，编译器任务的输出略有不同，但也包括完整的类名和方法。

## C.5 进程区域格式

进程部分在线程部分之后打印。它包含有关整个进程的信息，包括进程的线程列表和内存使用情况。

### C.5.1 线程列表

线程列表包括VM知道的线程。 这包括所有Java线程和一些VM内部线程，但不包括未附加到VM的用户应用程序创建的任何本地线程。 输出格式如下。
```
=>0x0805ac88 JavaThread "main" [_thread_in_native, id=21139]
|      |         |        |             |             +----- ID
|      |         |        |             +------------------- state
|      |         |        |                                  (JavaThread only)
|      |         |        +--------------------------------- name
|      |         +------------------------------------------ type
|      +---------------------------------------------------- pointer
+------------------------------------------------------ "=>" current thread
```
下面是一个输出示例：
```
Java Threads: ( => current thread )
  0x080c8da0 JavaThread "Low Memory Detector" daemon [_thread_blocked, id=21147]
  0x080c7988 JavaThread "CompilerThread0" daemon [_thread_blocked, id=21146]
  0x080c6a48 JavaThread "Signal Dispatcher" daemon [_thread_blocked, id=21145]
  0x080bb5f8 JavaThread "Finalizer" daemon [_thread_blocked, id=21144]
  0x080ba940 JavaThread "Reference Handler" daemon [_thread_blocked, id=21143]
=>0x0805ac88 JavaThread "main [_thread_in_native, id=21139]
Other Threads:
  0x080b6070 VMThread [id=21142]
  0x080ca088 WatcherThread [id=21148]
```
有关线程类型和线程状态的描述请见[C.4 Thread Section Format](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/felog.html#gbvki).

### C.5.2 VM状态

接下来的信息是VM状态，描述了整个虚拟机的状态。下面的表格描述了VM的一般状态。

| **VM一般状态** | **描述** |
| --- | --- |
| not at a safepoint | 正常执行 |
| at safepoint | 所有线程被阻塞在VM中以等待特殊VM操作完成 |
| synchronizing | 请求了一个特殊的VM操作，VM正在等待VM中所有的线程阻塞 |

VM状态输出是错误日志中的单行输出，如下：
```
VM state:not at safepoint (normal execution)
```
#### **C.5.3 互斥锁和监视器**

错误日志中的下一个信息是当前由线程拥有的互斥锁和监视器的列表。 这些互斥锁是VM内部锁，而不是与Java对象关联的监视器。 下面的示例显示了在保持VM锁定时发生crash时输出的外观。对于每个锁，日志包含锁的名称，其所有者以及VM内部互斥结构的地址及其操作系统锁。通常，此信息仅对非常熟悉HotSpot VM的用户有用。 所有者线程可以交叉引用到线程列表。
```
VM Mutex/Monitor currently owned by a thread:

([mutex/lock_event])[0x007357b0/0x0000031c] Threads_lock - owner thread: 0x00996318

[0x00735978/0x000002e0] Heap_lock - owner thread: 0x00736218
```
### C.5.4 堆摘要

下一个信息是堆的摘要。 输出取决于垃圾收集（GC）配置。 在此示例中，使用串行收集器，禁用类数据共享，并且tenured generation为空。 这可能表示致命错误发生在早期或启动期间，并且GC尚未将任何对象提升为终身代。 以下是此输出的示例。
```
Heap
 def new generation   total 576K, used 161K [0x46570000, 0x46610000, 0x46a50000)
  eden space 512K,  31% used [0x46570000, 0x46598768, 0x465f0000)
  from space 64K,   0% used [0x465f0000, 0x465f0000, 0x46600000)
  to   space 64K,   0% used [0x46600000, 0x46600000, 0x46610000)
 tenured generation   total 1408K, used 0K [0x46a50000, 0x46bb0000, 0x4a570000)
   the space 1408K,   0% used [0x46a50000, 0x46a50000, 0x46a50200, 0x46bb0000)
 compacting perm gen  total 8192K, used 1319K [0x4a570000, 0x4ad70000, 0x4e570000)
   the space 8192K,  16% used [0x4a570000, 0x4a6b9d48, 0x4a6b9e00, 0x4ad70000)
No shared spaces configured.
```
### C.5.5 内存映射

日志中的下一个信息是crash时的虚拟内存区域列表。对于大型应用程序，此列表可能很长。调试一些crash时，内存映射非常有用，因为它可以告诉你实际使用的库，它们在内存中的位置，以及堆，堆栈和保护页的位置。

内存映射的格式是特定于操作系统的。 在Solaris操作系统上，将打印基本地址和库名称。 在Linux系统上，打印进程内存映射（/proc/pid/maps）。 在Windows系统上，将打印每个库的基址和结束地址。 在Linux/x86上生成以下示例输出。 注意，为了简洁起见，示例中省略了大多数行。
```
Dynamic libraries:
08048000-08056000 r-xp 00000000 03:05 259171     /h/jdk6/bin/java
08056000-08058000 rw-p 0000d000 03:05 259171     /h/jdk6/bin/java
08058000-0818e000 rwxp 00000000 00:00 0
40000000-40013000 r-xp 00000000 03:0a 400046     /lib/ld-2.2.5.so
40013000-40014000 rw-p 00013000 03:0a 400046     /lib/ld-2.2.5.so
40014000-40015000 r--p 00000000 00:00 0
Lines omitted.
4123d000-4125a000 rwxp 00001000 00:00 0
4125a000-4125f000 rwxp 00000000 00:00 0
4125f000-4127b000 rwxp 00023000 00:00 0
4127b000-4127e000 ---p 00003000 00:00 0
4127e000-412fb000 rwxp 00006000 00:00 0
412fb000-412fe000 ---p 00083000 00:00 0
412fe000-4137b000 rwxp 00086000 00:00 0
Lines omitted.
44600000-46570000 rwxp 00090000 00:00 0
46570000-46610000 rwxp 00000000 00:00 0
46610000-46a50000 rwxp 020a0000 00:00 0
46a50000-46bb0000 rwxp 00000000 00:00 0
46bb0000-4a570000 rwxp 02640000 00:00 0
Lines omitted.
```
上述内存映射中的行的格式如下：
```
40049000-4035c000 r-xp 00000000 03:05 824473      /jdk1.5/jre/lib/i386/client/libjvm.so
|<------------->|  ^      ^       ^     ^        |<----------------------------------->|
  Memory region    |      |       |     |                           |
                   |      |       |     |                           |
  Permission   --- +      |       |     |                           |
    r: read               |       |     |                           |
    w: write              |       |     |                           |
    x: execute            |       |     |                           |
    p: private            |       |     |                           |
    s: share              |       |     |                           |
                          |       |     |                           |
  File offset   ----------+       |     |                           |
                                  |     |                           |
  Major ID and minor ID of -------+     |                           |
  the device where the file             |                           |
  is located (i.e. /dev/hda5)           |                           |
                                        |                           |
  inode number  ------------------------+                           |
                                                                    |
  File name  -------------------------------------------------------+
```
在内存映射输出中，每个库都有两个虚拟内存区域：一个用于代码，另一个用于数据。 代码段的权限标记为r-xp（可读，可执行，私有），数据段的权限为rw-p（可读，可写，私有）。

Java堆已经包含在输出中的堆摘要中，但是验证为堆保留的实际内存区域是否与堆摘要中的值匹配以及属性是否设置为rwxp可能很有用。

线程堆栈通常在内存映射中显示为两个背对背区域，一个具有权限--- p（保护页面），另一个具有权限rwxp（实际堆栈空间）。 此外，了解防护页面大小或堆栈大小很有用。 例如，在该存储器映射中，堆栈位于4127b000到412fb000。

在Windows系统上，内存映射输出是每个已加载模块的加载和结束地址，如下例所示。
```
Dynamic libraries:
0x00400000 - 0x0040c000     c:\jdk6\bin\java.exe
0x77f50000 - 0x77ff7000     C:\WINDOWS\System32\ntdll.dll
0x77e60000 - 0x77f46000     C:\WINDOWS\system32\kernel32.dll
0x77dd0000 - 0x77e5d000     C:\WINDOWS\system32\ADVAPI32.dll
0x78000000 - 0x78087000     C:\WINDOWS\system32\RPCRT4.dll
0x77c10000 - 0x77c63000     C:\WINDOWS\system32\MSVCRT.dll
0x08000000 - 0x08183000     c:\jdk6\jre\bin\client\jvm.dll
0x77d40000 - 0x77dcc000     C:\WINDOWS\system32\USER32.dll
0x7e090000 - 0x7e0d1000     C:\WINDOWS\system32\GDI32.dll
0x76b40000 - 0x76b6c000     C:\WINDOWS\System32\WINMM.dll
0x6d2f0000 - 0x6d2f8000     c:\jdk6\jre\bin\hpi.dll
0x76bf0000 - 0x76bfb000     C:\WINDOWS\System32\PSAPI.DLL
0x6d680000 - 0x6d68c000     c:\jdk6\jre\bin\verify.dll
0x6d370000 - 0x6d38d000     c:\jdk6\jre\bin\java.dll
0x6d6a0000 - 0x6d6af000     c:\jdk6\jre\bin\zip.dll
0x10000000 - 0x10032000     C:\bugs\crash2\App.dll
```
### C.5.6 VM参数和环境变量

错误日志中的下一个信息是VM参数列表，后跟环境变量列表。 例子如下：
```
VM Arguments:
java_command: NativeSEGV 2
Environment Variables:
JAVA_HOME=/h/jdk
PATH=/h/jdk/bin:.:/h/bin:/usr/bin:/usr/X11R6/bin:/usr/local/bin:
     /usr/dist/local/exe:/usr/dist/exe:/bin:/usr/sbin:/usr/ccs/bin:
     /usr/ucb:/usr/bsd:/usr/etc:/etc:/usr/dt/bin:/usr/openwin/bin:
     /usr/sbin:/sbin:/h:/net/prt-web/prt/bin
USERNAME=user
LD_LIBRARY_PATH=/h/jdk6/jre/lib/i386/client:/h/jdk6/jre/lib/i386:
     /h/jdk6/jre/../lib/i386:/h/bugs/NativeSEGV
SHELL=/bin/tcsh
DISPLAY=:0.0
HOSTTYPE=i386-linux
OSTYPE=linux
ARCH=Linux
MACHTYPE=i386
```
请注意，环境变量列表不是完整列表，而是适用于Java VM的环境变量的子集。

### C.5.7 信号处理程序

在Solaris OS和Linux上，错误日志中的下一个信息是信号处理程序列表。
```
Signal Handlers:
SIGSEGV: [libjvm.so+0x3aea90], sa_mask[0]=0xfffbfeff, sa_flags=0x10000004
SIGBUS: [libjvm.so+0x3aea90], sa_mask[0]=0xfffbfeff, sa_flags=0x10000004
SIGFPE: [libjvm.so+0x304e70], sa_mask[0]=0xfffbfeff, sa_flags=0x10000004
SIGPIPE: [libjvm.so+0x304e70], sa_mask[0]=0xfffbfeff, sa_flags=0x10000004
SIGILL: [libjvm.so+0x304e70], sa_mask[0]=0xfffbfeff, sa_flags=0x10000004
SIGUSR1: SIG_DFL, sa_mask[0]=0x00000000, sa_flags=0x00000000
SIGUSR2: [libjvm.so+0x306e80], sa_mask[0]=0x80000000, sa_flags=0x10000004
SIGHUP: [libjvm.so+0x3068a0], sa_mask[0]=0xfffbfeff, sa_flags=0x10000004
SIGINT: [libjvm.so+0x3068a0], sa_mask[0]=0xfffbfeff, sa_flags=0x10000004
SIGQUIT: [libjvm.so+0x3068a0], sa_mask[0]=0xfffbfeff, sa_flags=0x10000004
SIGTERM: [libjvm.so+0x3068a0], sa_mask[0]=0xfffbfeff, sa_flags=0x10000004
SIGUSR2: [libjvm.so+0x306e80], sa_mask[0]=0x80000000, sa_flags=0x10000004
```
## C.6 系统区域格式

错误日志中的最后一部分是系统信息。 输出是特定于操作系统的，但通常包括操作系统版本，CPU信息和有关内存配置的摘要信息。

以下示例显示Solaris 9 OS系统上的输出。
```
---------------  S Y S T E M  ---------------
OS:                       Solaris 9 12/05 s9s_u5wos_08b SPARC
           Copyright 2005 Sun Microsystems, Inc.  All Rights Reserved.
                        Use is subject to license terms.
                           Assembled 21 November 2005
uname:SunOS 5.9 Generic_112233-10 sun4u  (T2 libthread)
rlimit: STACK 8192k, CORE infinity, NOFILE 65536, AS infinity
load average:0.41 0.14 0.09
CPU:total 2 has_v8, has_v9, has_vis1, has_vis2, is_ultra3

Memory: 8k page, physical 2097152k(1394472k free)
vm_info: Java HotSpot(TM) Client VM (1.5-internal) for solaris-sparc,

built on Aug 12 2005 10:22:32 by unknown with unknown Workshop:0x550
```
在Solaris OS和Linux上，操作系统信息包含在文件/etc/\*发行版中。此文件描述了运行应用程序的系统类型，在某些情况下，信息字符串可能包含修补程序级别。某些系统升级未反映在/etc/\*发布文件中。在Linux系统上尤其如此，用户可以在其中重建系统的任何部分。

在Solaris OS上，uname系统调用用于获取内核的名称。 还会打印线程库（T1或T2）。

在Linux系统上，uname系统调用也用于获取内核名称。 还会打印libc版本和线程库类型。 一个例子如下。
```
uname:Linux 2.4.18-3smp #1 SMP Thu Apr 18 07:27:31 EDT 2002 i686

libc:glibc 2.2.5 stable linuxthreads (floating stack)

    |<- glibc version ->|<--  pthread type       -->|
```
在Linux上有三种可能的线程类型，即linuxthreads（固定栈），linuxthreads（浮动栈）和NPTL。它们通常安装在/lib，/lib/i686和/lib/tls中。

了解线程类型很有用。例如，如果crash似乎与pthread有关，那么你可以通过选择不同的pthread库来解决问题。可以通过设置LD_LIBRARY_PATH或LD_ASSUME_KERNEL来选择不同的pthread库（和libc）。

glibc版本通常不包括补丁级别。 rpm -q glibc命令可能会提供更详细的版本信息。

在Solaris OS和Linux上，下一个信息是rlimit信息。请注意，VM的默认栈大小通常小于系统限制，例子如下。
```
rlimit: STACK 8192k, CORE 0k, NPROC 4092, NOFILE 1024, AS infinity
             |          |         |           |           virtual memory (-v)
             |          |         |           +--- max open files (ulimit -n)
             |          |         +----------- max user processes (ulimit -u)
             |          +------------------------- core dump size (ulimit -c)
             +---------------------------------------- stack size (ulimit -s)
load average:0.04 0.05 0.02
```
下一个信息指定VM在启动时标识的CPU体系结构和功能，如以下示例所示。

CPU:total 2 family 6, cmov, cx8, fxsr, mmx, sse | | |<----- CPU features ---->| | | | +--- processor family (IA32 only): | 3 - i386 | 4 - i486 | 5 - Pentium | 6 - PentiumPro, PII, PIII | 15 - Pentium 4 +------------ Total number of CPUs

下表显示了SPARC系统上可能的CPU功能。

#### SPARC Features

| **SPARC Feature** | **Description** |
| --- | --- |
| has\_v8 | Supports v8 instructions.支持第8版指令 |
| has\_v9 | Supports v9 instructions.支持第9版指令 |
| has\_vis1 | Supports visualization instructions.支持可视化指令 |
| has\_vis2 | Supports visualization instructions.支持可视化指令 |
| is\_ultra3 | UltraSparc III. |
| no-muldiv | No hardware integer multiply and divide.没有硬件整数乘法和除法 |
| no-fsmuld | No multiply-add and multiply-subtract instructions.没有乘加和乘减指令 |


下表显示了Inter/IA32系统上可能的CPU功能。

#### Intel/IA32 Features

| **Intel/IA32 Feature** | **Description** |
| --- | --- |
| cmov | Supports cmov instruction.支持cmov指令 |
| cx8 | Supports cmpxchg8b instruction.支持cmpxchg8b指令 |
| fxsr | Supports fxsave and fxrstor.支持fssave和fxrstor |
| mmx | Supports MMX.支持MMX |
| sse | Supports SSE extensions.支持SSE扩展 |
| sse2 | Supports SSE2 extensions.支持SSE2扩展 |
| ht | Supports Hyper-Threading Technology.支持超线程技术 |

下表显示了AMD64/EM64T系统上可能的CPU功能。

#### AMD64/EM64T Features
| **AMD64/EM64T Feature** | **Description** |
| --- | --- |
| amd64 | AMD Opteron, Athlon64, and so forth.AMD Opteron, Athlon64等 |
| em64t | Intel EM64T processor.Inter EM64T处理器 |
| 3dnow | Supports 3DNow extension.支持3DNow扩展 |
| ht | Supports Hyper-Threading Technology.支持超线程技术 |


错误日志中下一个信息是内存信息，如下：
```
                                                        unused swap space
                                total amount of swap space      |
                      unused physical memory         |          |
  total amount of physical memory     |              |          |
       page size            |         |              |          |
           v                v         v              v          v
Memory: 4k page, physical 513604k(11228k free), swap 530104k(497504k free)
```
某些系统要求交换空间至少是实际物理内存大小的两倍，而其他系统则没有任何此类要求。作为一般规则，如果物理内存和交换空间几乎都已满，则有充分的理由怀疑crash是由于内存不足造成的。

在Linux系统上，内核可以将大多数未使用的物理内存转换为文件缓存。当需要更多内存时，Linux内核会将缓存内存返回给应用程序。这由内核透明地处理，但它确实意味着当仍有足够的可用物理内存时，致命错误处理程序报告的未使用物理内存量可能接近于零。

错误日志的SYSTEM部分中的最终信息是vm_info，它是嵌入在libjvm.so/jvm.dll中的版本字符串。 每个Java VM都有自己唯一的vm_info字符串。 如果你对特定Java VM是否生成致命错误日志有疑问，请检查版本字符串。
