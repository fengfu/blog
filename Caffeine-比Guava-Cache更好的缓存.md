---
title: Caffeine-比Guava Cache更好的缓存
date: 2018-03-26 17:09:15
tags: 性能
---

## 前言 ##
说起Guava Cache，很多人都不会陌生，它是Google Guava工具包中的一个非常方便易用的本地化缓存实现，基于LRU算法实现，支持多种缓存过期策略。由于Guava的大量使用，Guava Cache也得到了大量的应用。但是，Guava Cache的性能一定是最好的吗？也许，曾经，它的性能是非常不错的。但所谓长江后浪推前浪，总会有更加优秀的技术出现。今天，我就来介绍一个比Guava Cache性能更高的缓存框架：Caffeine。

下面的内容是转载的一篇译文，如果需要查看译文原文，请点击[这里](https://segmentfault.com/a/1190000008751999),英语好的同学也可以直接查看[英文原作](http://highscalability.com/blog/2016/1/25/design-of-a-modern-cache.html)。

缓存是提升性能的通用方法，现在大多数的缓存实现都使用了经典的技术。这篇文章中，我们会发掘[Caffeine](https://github.com/ben-manes/caffeine)中的现代的实现方法。Caffeine是一个开源的Java缓存库，它能提供高命中率和出色的并发能力。期望读者们能被这些想法激发，进而将它们应用到任何你喜欢的编程语言中。

## 驱逐策略 ##

缓存的驱逐策略是为了预测哪些数据在短期内最可能被再次用到，从而提升缓存的命中率。由于简洁的实现、高效的运行时表现以及在常规的使用场景下有不错的命中率，LRU（Least Recently Used）策略或许是最流行的驱逐策略。但LRU通过历史数据来预测未来是局限的，它会认为最后到来的数据是最可能被再次访问的，从而给与它最高的优先级。

现代缓存扩展了对历史数据的使用，结合就近程度（recency）和访问频次（frequency）来更好的预测数据。其中一种保留历史信息的方式是使用popularity sketch（一种压缩、概率性的数据结构）来从一大堆访问事件中定位频繁的访问者。可以参考[CountMin Sketch](http://dimacs.rutgers.edu/~graham/pubs/papers/cmsoft.pdf)算法，它由计数矩阵和多个哈希方法实现。发生一次读取时，矩阵中每行对应的计数器增加计数，估算频率时，取数据对应是所有行中计数的最小值。这个方法让我们从空间、效率、以及适配矩阵的长宽引起的哈希碰撞的错误率上做权衡。

![CountMin Sketch](https://sfault-image.b0.upaiyun.com/198/843/1988435212-58d64be00bb9d_articlex)

[Window TinyLFU](https://arxiv.org/pdf/1512.00727.pdf)（W-TinyLFU）算法将sketch作为过滤器，当新来的数据比要驱逐的数据高频时，这个数据才会被缓存接纳。这个许可窗口给予每个数据项积累热度的机会，而不是立即过滤掉。这避免了持续的未命中，特别是在突然流量暴涨的的场景中，一些短暂的重复流量就不会被长期保留。为了刷新历史数据，一个时间衰减进程被周期性或增量的执行，给所有计数器减半。

![Window TinyLFU](https://sfault-image.b0.upaiyun.com/274/307/2743073853-58d64be0088fe_articlex)

对于长期保留的数据，W-TinyLFU使用了分段LRU（Segmented LRU，缩写SLRU）策略。起初，一个数据项存储被存储在试用段（probationary segment）中，在后续被访问到时，它会被提升到保护段（protected segment）中（保护段占总容量的80%）。保护段满后，有的数据会被淘汰回试用段，这也可能级联的触发试用段的淘汰。这套机制确保了访问间隔小的热数据被保存下来，而被重复访问少的冷数据则被回收。

![](https://sfault-image.b0.upaiyun.com/571/159/571159220-58d64be031155_articlex)

如图中数据库和搜索场景的结果展示，通过考虑就近程度和频率能大大提升LRU的表现。一些高级的策略，像ARC，LIRS和W-TinyLFU都提供了接近最理想的命中率。想看更多的场景测试，请查看相应的论文，也可以在使用simulator来测试自己的场景。

## 过期策略 ##
过期的实现里，往往每个数据项拥有不同的过期时间。因为容量的限制，过期后数据需要被懒淘汰，否则这些已过期的脏数据会污染到整个缓存。一般缓存中会启用专有的清扫线程周期性的遍历清理缓存。这个策略相比在每次读写操作时按照过期时间排序的优先队列来清理过期缓存要好，因为后台线程隐藏了的过期数据清除的时间开销。

鉴于大多数场景里不同数据项使用的都是固定的过期时长，Caffien采用了统一过期时间的方式。这个限制让用O（1）的有序队列组织数据成为可能。针对数据的写后过期，维护了一个写入顺序队列，针对读后过期，维护了一个读取顺序队列。缓存能复用驱逐策略下的队列以及下面将要介绍的并发机制，让过期的数据项在缓存的维护阶段被抛弃掉。

## 并发 ##
由于在大多数的缓存策略中，数据的读取都会伴随对缓存状态的写操作，并发的缓存读取被视为一个难点问题。传统的解决方式是用同步锁。这可以通过将缓存的数据划成多个分区来进行锁拆分优化。不幸的是热点数据所持有的锁会比其他数据更常的被占有，在这种场景下锁拆分的性能提升也就没那么好了。当单个锁的竞争成为瓶颈后，接下来的经典的优化方式是只更新单个数据的元数据信息，以及使用[随机采样](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.110.8469&rep=rep1&type=pdf)、[基于FIFO](https://en.wikipedia.org/wiki/Page_replacement_algorithm#Second-chance)的驱逐策略来减少数据操作。这些策略会带来高性能的读和低性能的写，同时在选择驱逐对象时也比较困难。

另一种[可行方案](http://web.cse.ohio-state.edu/hpcs/WWW/HTML/publications/papers/TR-09-1.pdf)来自于数据库理论，通过提交日志的方式来扩展写的性能。写入操作先记入日志中，随后异步的批量执行，而不是立即写入到数据结构中。这种思想可以应用到缓存中，执行哈希表的操作，将操作记录到缓冲区，然后在合适的时机执行缓冲区中的内容。这个策略依然需要同步锁或者tryLock，不同的是把对锁的竞争转移到对缓冲区的追加写上。

在Caffeine中，有一组缓冲区被用来记录读写。一次访问首先会被因线程而异的哈希到stripped ring buffer上，当检测到竞争时，缓冲区会自动扩容。一个ring buffer容量满载后，会触发异步的执行操作，而后续的对该ring buffer的写入会被丢弃，直到这个ring buffer可被使用。虽然因为ring buffer容量满而无法被记录该访问，但缓存值依然会返回给调用方。这种策略信息的丢失不会带来大的影响，因为W-TinyLFU能识别出我们希望保存的热点数据。通过使用因线程而异的哈希算法替代在数据项的键上做哈希，缓存避免了瞬时的热点key的竞争问题。

![](https://sfault-image.b0.upaiyun.com/270/327/2703271825-58d64be011a4f_articlex)

写数据时，采用更传统的并发队列，每次变更会引起一次立即的执行。虽然数据的损失是不可接受的，但我们仍然有很多方法可以来优化写缓冲区。所有类型的缓冲区都被多个的线程写入，但却通过单个线程来执行。这种多生产者/单个消费者的模式允许了更简单、高效的算法来实现。

缓冲区和细粒度的写带来了单个数据项的操作乱序的竞态条件。插入、读取、更新、删除都可能被各种顺序的重放，如果这个策略控制的不合适，则可能引起悬垂索引。解决方案是通过状态机来定义单个数据项的生命周期。

![](https://sfault-image.b0.upaiyun.com/270/327/2703271825-58d64be011a4f_articlex)

在[基准测试](https://github.com/ben-manes/caffeine/wiki/Benchmarks#read-100-1)中，缓冲区随着哈希表的增长而增长，它的的使用相对更节省资源。读的性能随着CPU的核数线性增长，是哈希表吞吐量的33%。写入有10%的性能损耗，这是因为更新哈希表时的竞争是最主要的开销。

![](https://sfault-image.b0.upaiyun.com/250/918/2509180021-58d64be00f7f4_articlex)

## 结论 ##
还有许多实用的话题没有被覆盖到。包括最小化内存的技巧，当复杂度上升时保证质量的测试技术以及确定优化是否值得的性能分析方法。这些都是缓存的实践者需要关注的点，因为一旦这些被忽视，就很难重拾掌控缓存带来的复杂度的信心。

Caffeine的设计实现来自于大量的洞见和许多贡献者的共同努力。它这些年的演化离不开一些人的帮助：Charles Fry, Adam Zell, Gil Einziger, Roy Friedman, Kevin Bourrillion, Bob Lee, Doug Lea, Josh Bloch, Bob Lane, Nitsan Wakart, Thomas Müeller, Dominic Tootell, Louis Wasserman, and Vladimir Blagojevic. Thanks to Nitsan Wakart, Adam Zell, Roy Friedman, and Will Chu for their feedback on drafts of this article.
