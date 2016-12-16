---
title: Java Instrumentation研究之动态植入
date: 2016-04-25 10:20:43
tags: Java
---

## 1.前言 ##
前一篇文章我们提到：在Java SE5中，Instrument要求在运行前利用命令行参数或者系统参数来设置代理类，在实际的运行之中，虚拟机在初始化之时（在绝大多数的 Java 类库被载入之前），instrumentation的设置已经启动，并在虚拟机中设置了回调函数，检测特定类的加载情况，并完成实际工作。也就是说，在Java SE5中，我们只能使用premain的方式实现Instrumentation。但是在实际的很多的情况下，我们没有办法在虚拟机启动之时就为其设定代理，这样实际上限制了instrument的应用。而Java SE6的新特性改变了这种情况，通过agentmain和Java Tool API中的attach方式，我们可以很方便地在运行过程中动态地设置加载代理类，以达到instrumentation的目的。本文就是对Java动态Instrumentation的实现进行研究。

## 2.关于agentmain ##
在Java SE6的Instrumentation 当中，有一个跟 premain“并驾齐驱”的“agentmain”方法，可以在main函数开始运行之后再运行。
跟premain函数一样， 开发者可以编写一个含有“agentmain”函数的Java类：

    public static void agentmain(String agentArgs, Instrumentation inst); 		 [1] 
	public static void agentmain(String agentArgs); 			 [2]

[1]的优先级比[2]高，将会被优先执行。

跟前文提到的premain函数一样，我们可以在agentmain方法中对类进行各种操作。其中的agentArgs和Inst的用法跟premain相同。
与“Premain-Class”类似，我们必须在manifest文件里面设置“Agent-Class”来指定包含agentmain函数的类。
可是，跟premain不同的是，agentmain需要在main函数开始运行后才启动，这样的时机应该如何确定呢，这样的功能又如何实现呢？
在 Java SE6 文档当中，我们也许无法在java.lang.instrument包相关的文档部分看到明确的介绍，更加无法看到具体的应用agnetmain的例子。不过，在 Java SE6的新特性里面，有一个不太起眼的地方，揭示了agentmain的用法。这就是Java SE6当中提供的Attach API。

## 3.关于Attach API ##
[Attach API](http://docs.oracle.com/javase/7/docs/jdk/api/attach/spec/index.html)不是Java的标准API，而是Sun公司提供的一套扩展API，用来向目标JVM“附着”（Attach）代理工具程序的。有了它，我们可以方便地监控一个JVM，运行一个外加的代理程序。
Attach API很简单，只有2个主要的类，都在com.sun.tools.attach包里面：VirtualMachine代表一个Java虚拟机，也就是程序需要监控的目标虚拟机，提供了JVM枚举，Attach动作和Detach动作（Attach 动作的相反行为，从JVM 上面解除一个代理）等等 ; VirtualMachineDescriptor则是一个描述虚拟机的容器类，配合VirtualMachine类完成各种功能。

## 4.agentmain实现步骤 ##
与Permain类似，agentmain方式同样需要提供一个agent jar，并且这个jar需要满足：

1. 在manifest中指定Agent-Class属性，值为代理类全路径；
2. 代理类需要提供public static void agentmain(String args, Instrumentation inst)或public static void agentmain(String args)方法。并且再二者同时存在时以前者优先。args和inst和premain中的一致。

Attach API中的VirtualMachine代表一个运行中的VM，其提供了loadAgent()方法，可以在运行时动态加载一个代理jar，这样就可以实现类似premain的效果了。

## 5.agentmain实例 ##
与上文一样，我们要通过Java Instrumentation实现对某个类中所有方法执行时间的统计，只不过不同的是：这次我们采用动态Instrumentation的方式。

1. 编写一个简单的测试类Test，以便于后面我们通过动态代理实现方法计时功能；
2. 编写Agent实现类SampleTransformer，实现ClassFileTransformer接口。代码同上篇文章一样，这里不再赘述。
2. 编写Agent入口类DynamicAgent，实现agentmain方法：

    public static void agentmain(String args, Instrumentation inst)

1. 编写Agent加载类AgentLoader，以通过Attach API中的VirtualMachine来动态加载我们编写的代理jar；
2. 编写一个测试入口类SampleApp，能够长时间驻留，便于我们测试动态植入功能；本例中我们通过键盘数据接收用户的输入数据。只要用户输入不为“bye”，那么就会执行Test类的test()方法；在系统刚刚启动的时候，我们没有利用动态植入实现对test()方法的计时；当执行动态植入后，再执行test()方法，我们就能看到屏幕除了输出原test()方法输出的“hello world”之外，还额外输出了test()方法的运行计时。这就能够证明植入代码生效了；
3. 设置MANIFEST.MF文件，指定Agent-Class、Can-Retransform-Classes、Can-Redefine-Classes等属性。在本实例所付的代码中，是通过maven配置的，具体请参考附件中的代码；
4. 打包得到DynamicAgent-1.0-SNAPSHOT.jar；
5. 执行SampleApp，启动应用：

    java -cp /home/fengfu/DynamicAgent/target/DynamicAgent-1.0-SNAPSHOT.jar;/.m2/repository/org/javassist/javassist/3.19.0-GA/javassist-3.19.0-GA.jar io.fengfu.learning.instrument.SampleApp

在交互窗口输入test，我们可以看到系统输出：Hello World!

1. 得到SampleApp的进程id，比如1234；
2. 执行AgentLoader，启动动态代理：

    java -cp /home/fengfu/DynamicAgent/target/DynamicAgent-1.0-SNAPSHOT.jar;/usr/local/jdk1.7/lib/tools.jar;/.m2/repository/org/javassist/javassist/3.19.0-GA/javassist-3.19.0-GA.jar  io.fengfu.learning.instrument.AgentLoader 1234

1. 在SampleApp的交互窗口再次输入test，我们可以看到系统输出：Hello World!，同时也输出：Call to method test took 3001 ms.

大功告成，一个简单的动态植入功能就实现了~
本实例的代码可以点 [这里]((http://fengfu.io/attach/DynamicAgent.zip)) 下载。

## 6.说明 ##
实例虽然简单，但是在代码编写过程中还是遇到了几个坑，贴出来供大家参考：

1. 如果动态植入需要通过javassist去修改某些类，那么需要在MANIFEST.MF中设置Can-Retransform-Classes和Can-Redefine-Classes为true；
2. 要动态修改的类Test不要跟测试入口类SampleApp放在一个类文件中，否则代理启动动态修改Class时会报如下异常：

    	java.lang.reflect.InvocationTargetException
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(Unknown Source)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(Unknown Source)
        at java.lang.reflect.Method.invoke(Unknown Source)
        at sun.instrument.InstrumentationImpl.loadClassAndStartAgent(Unknown Source)
        at sun.instrument.InstrumentationImpl.loadClassAndCallAgentmain(Unknown Source)
		Caused by: java.lang.UnsupportedOperationException: class redefinition failed: attempted to add a method
        at sun.instrument.InstrumentationImpl.retransformClasses0(Native Method)
        at sun.instrument.InstrumentationImpl.retransformClasses(Unknown Source)
        at io.fengfu.learning.instrument.DynamicAgent.agentmain(DynamicAgent.java:14)
        ... 6 more

分析原因，是因为SampleApp正在运行中，而JVM是不允许reload一个正在运行时的类的。一旦classloader加载了一个class，在运行时就不能重新加载这个class的另一个版本。