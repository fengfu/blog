---
title: Java Instrumentation研究之premain
date: 2016-04-24 15:30:14
tags: Java
---

## 1.引言 ##
作为一个软件工程师，你有没有遇到过以下场景：

1. 为了排查线上问题，非常迫切地想要知道系统中某个方法的运行数据，为此只能临时修改代码，打印日志；
2. 为了查找系统瓶颈，你需要知道某个方法的耗时，为此你只能在方法头尾处记录时间戳，并打印日志或者记录到监控中；
3. ……

为了实现上述种种临时需求，你需要修改代码、发布、查看日志。或许，这些场景对于作为码农的你已经习以为常，但是世界是懒人创造的，那有没有一种方式能够避免我们这种重复又无任何积累的劳动呢？答案是有的。

最近，公司的大神 [余昭辉](http://www.cnblogs.com/yuyijq/) 发布了一个新的应用QTracer watch，可以支持通过简单的配置，查看系统运行时的数据。而实现这一点，竟然不用像以前一样在系统中添加一行代码并发布。这种高大上的技术实现立马让我对神的仰慕之情增加了几分。但仰慕归仰慕，对于神作我们还是需要知道其原理的。

通过了解，清楚了这高大上的技术实现来自于Java的Instrumentation(植入)。

## 2.Java Instrumentation简介 ##

Instrumentation是Java SE5的新特性，它把Java的instrument功能从本地代码中解放出来，使之可以用Java代码的方式解决问题。使用Instrumentation，开发者可以构建一个独立于应用程序的代理程序（Agent），用来监测和协助运行在JVM上的程序，甚至能够替换和修改某些类的定义。有了这样的功能，我们就可以实现更为灵活的运行时虚拟机监控和Java类操作了，这样的特性实际上提供了一种虚拟机级别支持的AOP实现方式，使得我们无需对原有代码做任何升级和改动，就可以实现某些 AOP 的功能了。
在Java SE6里面，instrument包被赋予了更强大的功能：启动后的instrument、本地代码（native code）instrument，以及动态改变classpath等。这些改变意味着Java具有了更强的动态控制、解释能力，使得Java语言变得更加灵活多变。
在Java SE6里面，最大的改变是运行时的Instrumentation成为可能。在Java SE5中，Instrument要求在运行前利用命令行参数或者系统参数来设置代理类，在实际的运行之中，虚拟机在初始化之时（在绝大多数的 Java 类库被载入之前），instrumentation的设置已经启动，并在虚拟机中设置了回调函数，检测特定类的加载情况，并完成实际工作。但是在实际的很多的情况下，我们没有办法在虚拟机启动之时就为其设定代理，这样实际上限制了instrument的应用。而Java SE 6的新特性改变了这种情况，通过Java Tool API中的attach方式，我们可以很方便地在运行过程中动态地设置加载代理类，以达到 instrumentation的目的。
另外，对native的Instrumentation也是Java SE6的一个崭新的功能，这使以前无法完成的功能--对native接口的instrumentation。我们可以在Java SE6中，通过一个或者一系列的prefix添加而得以完成。
最后，Java SE6里的Instrumentation也增加了动态添加classpath的功能。所有这些新的功能，都使得instrument包的功能更加丰富，从而使Java语言本身更加强大。

## 3.Java Instrumentation原理 ##

“java.lang.instrument”包的具体实现，依赖于JVMTI。JVMTI（Java Virtual Machine Tool Interface）是一套由 Java虚拟机提供的，为JVM相关的工具提供的本地编程接口集合。JVMTI是从Java SE5开始引入，整合和取代了以前使用的 Java Virtual Machine Profiler Interface (JVMPI) 和 the Java Virtual Machine Debug Interface (JVMDI)，而在Java SE6中，JVMPI和JVMDI已经消失了。JVMTI提供了一套“代理”程序机制，可以支持第三方工具程序以代理的方式连接和访问JVM，并利用JVMTI提供的丰富的编程接口，完成很多跟JVM相关的功能。事实上，java.lang.instrument包的实现，也就是基于这种机制的：在Instrumentation的实现当中，存在一个JVMTI的代理程序，通过调用JVMTI当中Java类相关的函数来完成Java类的动态操作。除开Instrumentation功能外，JVMTI还在虚拟机内存管理，线程控制，方法和变量操作等等方面提供了大量有价值的函数，具体可以参考[JVMTI官方文档](http://docs.oracle.com/javase/7/docs/platform/jvmti/jvmti.html)。

## 4.Java Instrumentation实现步骤 ##

在Java SE5时代，Instrument只提供了premain（命令行）一种方式，即在真正的应用程序（包含main方法的程序）main方法启动前启动一个代理程序。而在Java SE6中则包含两种应用Instrumentation的方式：premain（命令行）和agentmain（运行时）。在本文中，我们首先研究premain方式。
要实现premain方式，我们要遵循的步骤如下：
1）编写Agent实现类，实现ClassFileTransformer接口。ClassFileTransformer中声明了一个方法：

    public byte[] transform(
    ClassLoader loader, 
    String className, 
    Class cBR, 
    java.security.ProtectionDomain pD, 
    byte[] classfileBuffer) throws IllegalClassFormatException

通过这个方法，代理可以得到虚拟机载入的类的字节码（通过 classfileBuffer 参数）。代理的各种功能一般是通过操作这一串字节码得以实现的。

2）编写Agent入口类，实现premain方法：

    public static void premain(String agentArgs, Instrumentation inst)

3）打包Agent：将上述步骤1中声明的Java类打包成一个jar文件，并在META-INF/MANIFEST.MF文件中加入“Premain-Class”来指定此Java类（注意此处需要声明全路径）；

    Manifest-Version: 1.0
    Premain-Class: io.fengfu.learning.instrument.SampleAgent

最终我们打包得到SampleAgent.jar。

4）执行命令：

    java -javaagent:SampleAgent-1.0-SNAPSHOT.jar -cp /home/fengfu/SampleAgent/SampleAgent-1.0-SNAPSHOT.jar;/home/fengfu/SampleAgent/lib/javassist-3.19.0-GA.jar io.fengfu.learning.instrument.SampleApp

java选项中有-javaagent:xx，xx就是你的agent jar，java通过此选项加载agent,由agent来监控classpath下的应用。
如果你的Agent类引入别的包，那么需要使用-cp参数指定包的路径，否则在执行java命令时，会报找不到类的错误。

## 5.Java Instrumentation实例(premain) ##

本实例基于javassist实现了对某个类中所有方法执行时间的统计，具体代码请点击 [这里](http://fengfu.io/attach/SampleAgent.zip) 下载。