---
title: jcmd命令详解
date: 2016-12-14 16:28:59
tags: Java
---
# 简介 #
在JDK 1.7及之后的版本，新增了一个命令行工具jcmd。它是一个多功能工具，可以做哪些事情呢？这里先卖个关子，先来看看Oracle官方对jcmd的介绍:

*The jcmd utility is used to send diagnostic command requests to the JVM, where these requests are useful for controlling Java Flight Recordings, troubleshoot, and diagnose JVM and Java Applications. It must be used on the same machine where the JVM is running, and have the same effective user and group identifiers that were used to launch the JVM.*

翻译成人话就是：
jcmd一个用来发送诊断命令请求到JVM的工具，这些请求对于控制Java飞行记录器、故障排除、JVM和Java应用诊断来说是比较有用的。jcmd必须与正在运行的JVM在同一台机器上使用，并且使用启动该JVM时的用户权限。

jcmd的用法是:
`jcmd <process id/main class> <command> [options]`

其中<command>的说明如下：

|命令|说明|
|---|---|
|help	|	打印帮助信息，示例：jcmd <PID> help [<command name>]|
|ManagementAgent.stop	|	停止JMX Agent|
|ManagementAgent.start_local	|	开启本地JMX Agent|
|ManagementAgent.start	|	开启JMX Agent|
|Thread.print	|	参数-l打印java.util.concurrent锁信息，相当于：jstack <PID>|
|PerfCounter.print	|	相当于：jstat -J-Djstat.showUnsupported=true -snap <PID>|
|GC.class_histogram	|	相当于：jmap -histo <PID>|
|GC.heap_dump	|	相当于：jmap -dump:format=b,file=xxx.bin <PID>|
|GC.run_finalization	|	相当于：System.runFinalization()|
|GC.run	|	相当于：System.gc()|
|VM.uptime	|	参数-date打印当前时间，VM启动到现在的时候，以秒为单位显示|
|VM.flags	|	参数-all输出全部，相当于：jinfo -flags <PID>, jinfo -flag <VM FLAG> <PID>|
|VM.system_properties	|	相当于：jinfo -sysprops <PID>|
|VM.command_line	|	相当于：jinfo -sysprops <PID> | grep command|
|VM.version	|	相当于：jinfo -sysprops <PID> | grep version|
