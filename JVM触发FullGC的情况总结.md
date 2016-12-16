---
title: JVM触发FullGC的情况总结
date: 2016-03-08 22:53:56
tags: Java
comments: true
---

FullGC是Java应用中一个不容忽视的问题，因为FullGC会引起应用停顿，所以它对那些响应时间要求比较高的应用的影响还是非常大的。
当然，如果要解决FullGC的问题，我们首先需要知道在什么情况下会引起FullGC，这样才能对症下药，避免FullGC的出现。

下面的总结是针对于系统自动进行FullGC的情况分析，不包含System.gc操作。


## 1. 老年代空间不足##
JVM中堆空间主要由新生代和老年代组成。新创建的对象大多在新生代中创建，当对象经过几次Minor GC依然存活，才有机会被转入老年代。这时问题就来了，如果此时老年代的空间不足以容纳从新生代转入的对象，那么JVM就会进行FullGC，以清理老年代的空间。如果进行FullGC后老年代依然无法容纳转入对象，那么系统就会抛出：java.lang.OutOfMemoryError: Java heap space的异常，相比大家都比较熟悉了。


## 2. 永久代空间不足##
永久代(Permanent Generation)一般是用来存放类信息、字符串常量的地方，如果我们永久代设置的空间比较小无法容纳足够的类信息时，或者因为频繁热加载类信息，又或者存储了太多的字符串常量，那么系统就会触发FullGC，以清理永久代。如果FullGC之后永久代还无法容纳足够的信息，那么系统就会抛出：java.lang.OutOfMemoryError: PermGen space的异常，眼熟吧:)


## 3. promotion failed和concurrent mode failure##

promotion failed是指在进行Minor GC时，新生代中的对象从Eden区往survivor区转移，但是survivor区放不下，只能放入老年代，但是悲催的是老年代也放不下，这时就会出现promotion failed的情况，系统会触发FullGC以清理空间。

concurrent mode failure是指在进行CMS GC的过程中有对象要放入老年代，但是老年代空间不够引起的。 


## 4. 统计得到的Minor GC晋升到旧生代的平均大小大于旧生代的剩余空间##
这是一个较为复杂的触发情况，Hotspot为了避免由于新生代对象晋升到旧生代导致旧生代空间不足的现象，在进行Minor GC时，做了一个判断，如果之前统计所得到的Minor GC晋升到旧生代的平均大小大于旧生代的剩余空间，那么就直接触发Full GC。
例如程序第一次触发Minor GC后，有6MB的对象晋升到旧生代，那么当下一次Minor GC发生时，首先检查旧生代的剩余空间是否大于6MB，如果小于6MB，则执行Full GC。
当新生代采用PS GC时，方式稍有不同，PS GC是在Minor GC后也会检查，例如上面的例子中第一次Minor GC后，PS GC会检查此时旧生代的剩余空间是否大于6MB，如小于，则触发对旧生代的回收。
除了以上4种状况外，对于使用RMI来进行RPC或管理的Sun JDK应用而言，默认情况下会一小时执行一次Full GC。可通过在启动时通过- java -Dsun.rmi.dgc.client.gcInterval=3600000来设置Full GC执行的间隔时间或通过-XX:+ DisableExplicitGC来禁止RMI调用System.gc。

明确了FullGC出现的原理，我们就能根据JVM垃圾回收的情况来判断系统到底是因为什么原因出现了FullGC，也就能对症下药，避免FullGC的出现。