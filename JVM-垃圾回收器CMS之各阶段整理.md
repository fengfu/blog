---
title: JVM 垃圾回收器CMS之各阶段总结
date: 2016-06-21 15:50:22
tags: Java
---

先贴一张图：

![](https://img.alicdn.com/imgextra/i3/2657627814/TB2HFD0qVXXXXblXpXXXXXXXXXX_!!2657627814.jpg)

## 1.初始标记阶段(CMS-initial-mark) ##
  这个阶段的主要任务是找到堆中所有的垃圾回收根节点对象，这个阶段会暂停所有的应用程序线程，即STW(Stop the world)。此阶段会打印1行日志，如下：

	[GC [1 CMS-initial-mark: 2905437K(4096000K)] 3134625K(5916480K), 0.2551680 secs] [Times: user=0.26 sys=0.00, real=0.25 secs]

其中第一组数据，第一对数据标识老年代实际占用的空间大小和老年代分配的空间大小，第二对数据标识整个堆的实际使用情况和分配的堆的空间。
细心的人可能发现了CMS-initial-mark前面的数字“1”，这个标志代表了STW，在后面的“重新标记”阶段，你也会发现这个标志。这正好与上面图中说明的阶段是对应的。

## 2.标记阶段(CMS-concurrent-mark) ##
  这个阶段是和应用线程并发执行的，所谓并发收集器指的就是这个，主要作用是标记可达的对象。由于只是进行标记，所以不会对堆的占用产生实质的改变。

	2016-06-21T16:38:53.911+0800: 367848.849: [CMS-concurrent-mark-start]
	2016-06-21T16:38:57.241+0800: 367852.178: [CMS-concurrent-mark: 2.787/3.329 secs] [Times: user=12.12 sys=0.64, real=3.33 secs]
	
这个阶段会打印2行日志，第一行CMS-concurrent-mark-start标识标记阶段开始。第二行中的“2.787/3.329 secs”表示标记阶段的耗时。后面的“user=12.12”表示占用的cpu时间(此JVM运行的服务器CPU为4核)。

## 3.预清理阶段(CMS-concurrent-preclean) ##
  在之前的标记阶段，标记和应用线程是并发执行的，因此有些对象的状态在标记后会发生改变。这个阶段只要是发现从新生代晋升的对象、新分配到老年代的对象以及在标记阶段被修改了的对象。
  这个阶段也会打印2行日志，跟标记阶段类似。

```
2016-06-21T16:38:57.241+0800: 367852.178: [CMS-concurrent-preclean-start]
2016-06-21T16:38:57.718+0800: 367852.655: [CMS-concurrent-preclean: 0.342/0.477 secs] [Times: user=1.79 sys=0.10, real=0.48 secs]
```

## 4.重新标记阶段 ##
  可中断预清理阶段是在JDK1.5中加入的。这个阶段是CMS中比较复杂的一个阶段，因为在这个阶段，JVM会执行很多操作。
  首先是CMS-concurrent-abortable-preclean，可中断的预清理。我们可能要问，既然再上一个阶段已经执行了预清理了，为什么还要再做一次？我们知道，CMS是以获取最短停顿时间为目的的GC，所以简单说进行可中断预清理的目的就是希望尽量缩短停顿的时间。
  可中断预清理涉及几个参数：
  -XX:CMSMaxAbortablePrecleanTime：当abortable-preclean阶段执行达到这个时间时才会结束
  -XX:CMSScheduleRemarkEdenSizeThreshold（默认2m）：控制abortable-preclean阶段什么时候开始执行，即当eden使用达到此值时，才会开始abortable-preclean阶段
  -XX:CMSScheduleRemarkEdenPenetratio（默认50%）：控制abortable-preclean阶段什么时候结束执行

```
2016-06-21T16:38:57.718+0800: 367852.656: [CMS-concurrent-abortable-preclean-start]
2016-06-21T16:38:58.801+0800: 367853.738: [CMS-concurrent-abortable-preclean: 0.920/1.083 secs] [Times: user=4.06 sys=0.20, real=1.08 secs]
2016-06-21T16:38:58.808+0800: 367853.746: [GC[YG occupancy: 777901 K (1820480 K)]367853.746: [Rescan (parallel) , 0.1361120 secs]367853.882: [weak refs processing, 0.0005370 secs]367853.883: [scrub string table, 0.0044130 secs] [1 CMS-remark: 3034451K(4096000K)] 3812352K(5916480K), 0.1412750 secs] [Times: user=0.54 sys=0.00, real=0.14 secs]
```

  其次，是Rescan操作，此阶段暂停应用线程，对对象进行重新扫描并标记。通过上面的日志我们可以看到，在CMS-remark的时候有一次STW。另外，这个过程还会打印出弱引用处理、类卸载等过程的耗时。

## 5.清除阶段(CMS-concurrent-sweep) ##
  这个阶段开始进行垃圾的清理工作，此时应用线程被重新激活，回收线程与应用线程并发运行，那些无效的对象被清理掉。

```
2016-06-21T16:38:58.950+0800: 367853.888: [CMS-concurrent-sweep-start]
2016-06-21T16:39:05.850+0800: 367860.788: [CMS-concurrent-sweep: 5.656/6.900 secs] [Times: user=25.88 sys=1.28, real=6.90 secs]
```

## 6.初始标记阶段(CMS-concurrent-reset) ##
  这是CMS一个回收周期的最后一个阶段，在这个阶段，CMS会清除内部状态，为下次回收做准备。

```
2016-06-21T16:39:05.850+0800: 367860.788: [CMS-concurrent-reset-start]
2016-06-21T16:39:05.860+0800: 367860.798: [CMS-concurrent-reset: 0.010/0.010 secs] [Times: user=0.04 sys=0.00, real=0.01 secs]
```

  至此，一个CMS周期结束。