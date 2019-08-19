---
title: 高性能Java-集合
date: 2019-01-12 11:02:30
tags: Java
---
# 前言 #
集合是我们在编写代码过程中常用的数据类型。在Java中，常用的集合类型有List、Map和Set。本文将对一些常用的集合类型的特点进行分析，并针对一些会影响性能的注意事项进行说明。
# 1. 集合类型 #
## 1.1 List ##
### 1.1.1 ArrayList ##
顾名思义，基于数组实现，同时也说明容量是确定的，非线程安全。在不指定初始化容量的情况下，默认初始化的大小是10。当增加元素并且超出容量时，会以原容量50%的尺寸进行扩容，扩容时使用Arrays.copyOf(底层使用System.arraycopy)进行数组拷贝。
ArrayList适用于读多写少的场景，因为基于下标访问其中的元素，所以性能比较高。但在往指定位置add或者remove指定位置的元素时，由于涉及到指定位置后面元素的移动，会对性能造成影响。越是对前面位置元素的add/remove操作，对性能造成的影响越大。
### 1.2.2 LinkedList ###
基于双向链表实现，非线程安全。既然是链表，那么容量就可以是无限的了。
在按下标读取链表中的元素时，需要遍历部分链表进行指针移动，这样效率就比较低下了。但在add/remove指定位置的元素时，不再需要复制移动，只需修改前后节点的指针即可，所以适用于写多读少的场景。
这里扩展一下：在制定线程池的队列时，我们可以用ArrayBlockingQueue，也可以用LinkedBlockingQueue。那什么时候用LinkedBlockingQueue呢？答案是并发量比较大的时候，因为队列需要频繁入栈出栈，也就是写多读少，所以基于链表原理实现的LinkedBlockingQueue就比较适合了。
### 1.2.3 CopyOnWriteArrayList ###
线程安全的List。基于不可变对象策略，在修改时先复制出一个数组快照来修改，改好了，再让内部指针指向新数组。
因为对快照的修改对读操作来说不可见，所以读读之间不互斥，读写之间也不互斥，只有写写之间要加锁互斥。但复制快照的成本昂贵，典型的适合读多写少的场景。如果更新频率较高或数组较大时，还是得用Collections.synchronizedList(list)，对所有操作用加锁来保证线程安全。
## 1.2 Map ##
### 1.2.1 HashMap ###
基于拉链法实现，非线程安全。HashMap底层就是一个数组结构，数组中的每一项又是一个链表。当新建一个HashMap 的时候，就会初始化一个数组。
当我们往HashMap中put元素的时候，先根据key的hashCode重新计算hash值，根据hash值得到这个元素在数组中的位置（即下标），如果数组该位置上已经存放有其他元素了，那么在这个位置上的元素将以链表的形式存放，新加入的元素放在链头，最先加入的放在链尾。如果数组该位置上没有元素，就直接将该元素放到此数组中的该位置上。

当然，每个数组里最好只有一个元素，不用去比较。所以默认当Entry数量达到数组数量的75%时，哈希冲突已比较严重，就会成倍扩容数组，并重新分配所有原来的Entry。扩容成本不低（扩容成本、内存浪费），所以也最好有个预估值，这也就是为什么我们建议在初始化时指定初始容量的原因。
HashMap不是线程安全的，在JDK7及以前的版本中，在并发情况下修改HashMap容易导致死循环，JDK8以后修改了实现，虽然不会导致死循环，但在多线程情况下写依然不安全。
### 1.2.2 ConcurrentHashMap ###
线程安全的HashMap。JDK5~JDJ7的实现是用分段锁实现。默认16把写锁（可以设置更多），有效分散了阻塞的概率。数据结构为Segment[]，每个Segment一把锁。Segment里面才是哈希桶数组。Key先算出它在哪个Segment里，再去算它在哪个哈希桶里。
也没有读锁，因为put/remove动作是个原子动作（比如put的整个过程是一个对数组元素/Entry 指针的赋值操作），读操作不会看到一个更新动作的中间状态。
但在JDK8里，抛弃了分段锁技术的实现，直接采用Node（继承自Map.Entry）作为table元素，采用CAS + synchronized保证并发更新的安全性，底层采用数组+链表+红黑树的存储结构。修改时，不再采用ReentrantLock加锁，直接用内置synchronized加锁，Java8的内置锁比之前版本优化了很多，相较ReentrantLock，性能不并差。获取size时，使用CounterCell内部类，用于并行计算每个bucket的元素数量。
使用ConcurrentHashMap时，需要注意put和putIfAbsent的差别。
put:与之hashMap相同，当key存在时,put同样的key将被覆盖。
putIfAbsent:当key不存在的时候调用put方法将key存入进map。当key存在的时候相当于return map.get(key)。
### 1.2.3 LinkedHashMap ###
扩展HashMap，每个Entry增加双向链表，适用于写多读少的场景。
### 1.2.4 TreeMap ###
基于红黑树实现的排序Map，非线程安全。TreeMap的增删改查和统计相关的操作的时间复杂度都为 O(logn)。
由于TreeMap是基于红黑树的实现的排序Map，对于增删改查以及统计的时间复杂度都控制在O(logn)的级别上，相对于HashMap和LikedHashMap的统计操作的(最大的key，最小的key，大于某一个key的所有Entry等等)时间复杂度O(n)具有较高时间效率，所以TreeMap比较适合用于需要基于排序的统计功能。
## 1.3 Set ##
### 1.3.1 HashSet ###
内部是HashMap，非线程安全。
### 1.3.2 LinkedHashSet ###
内部是LinkedHashMap，非线程安全。
### 1.3.3 TreeSet ###
内部是TreeMap的SortedSet，非线程安全。
### 1.3.4 CopyOnWriteArraySet ###
基于CopyOnWriteArrayList实现，线程安全。
# 2. 一些框架 #
## 2.1 Koloboke ##
Koloboke是一个比较年轻的集合框架，为所有原始数据类型或对象的组合提供了HashMap和HashSet。在一些评测中，Koloboke取得了不错的成绩，号称实现最快也是存储最高效的库，但是它诞生不久并没有被广泛使用，所以存在一定的风险。
## 2.2 fastutil ##
一个意大利的计算机博士开发的集合库。fastutil中的集合数据类型有10种：8种基本数据类型+Object+Reference，这意味着你能够适用int、long这种基本数据类型作为map的key，同样也意味着这能够节省很多的存储空间。
## 2.3 Goldman Sachs Collections ##
高盛集团开源的集合框架，简称gs-collections。gs-collections出现时间比较早，也取得了比较广泛的应用。在性能评测中，gs-collections仅次于Koloboke，如果你正在寻找更加稳定和成熟的库，而不那么在意性能，你可以试试gs-collections。
## 2.4 Trove ##
Trove为所有的原始数据类型、对象的组合提供了链表、栈、队列、HashSet和HashMap。Trove是一个老牌的集合框架，经过几年的迭代已经比较稳定了，目前社区已经不太活跃了。
## 2.5 hppc ##
HPPC是High Performance Primitive Collections for Java的缩写，为所有原始数据类型提供了数组列表、数组队列、哈希集合和哈希映射。虽然号称高性能，但在测评中，性能只能说一般，所以这里不再详细介绍。
# 3. 一些注意事项 #
## 3.1 初始化容量 ##
集合类在不容量不足的情况下会进行扩容，而扩容操作会消耗一定的资源，所以在初始化集合的时候，尽量预估集合的尺寸并指定集合的初始化容量，这样能够避免集合频繁扩容带来的资源消耗。
## 3.2 遍历 ##
集合的遍历可以用for、foreach和iterator几种方式，那么这三者的速度有什么差异呢？答案是：
如果是ArrayList，用三种方式遍历的速度是for>Iterator>foreach，速度级别基本一致；这是因为ArrayList是基于索引(index)的数组，索引在数组中搜索和读取数据的时间复杂度是O(1)。
如果是LinkedList，则三种方式遍历的差距很大了,数据量大时越明显(一般是超过100000级别)，用for遍历的效率远远落后于foreach和Iterator，Iterator>foreach>>>for，因为LinkedList的底层实现则是一个双向循环带头节点的链表，因此LinkedList中插入或删除的时间复杂度仅为O(1)，但是获取数据的时间复杂度却是O(n)。
明白了两种List的区别之后，就知道，ArrayList用for循环随机读取的速度是很快的，因为ArrayList的下标是明确的，读取一个数据的时间复杂度仅为O(1)。但LinkedList若是用for来遍历效率很低，读取一个数据的时间复杂度就达到了为O(n)。而用Iterator的next()则是顺着链表节点顺序读取数据的效率就很高了。
综上： 
1. ArrayList用三种遍历方式都差得不算太多，一般都会用for或者foreach，因为Iterator写法相对复杂一些；
2. LinkedList的话，推荐使用foreach或者Iterator(数据量越大，三者方法差别明显)。 
## 3.3 排序 ##
使用堆排序(对于数组)或合并排序(对于双向链表)或快速排序(对于数组，但是你需要找个好的参考值)，不要使用选择排序、冒泡排序或插入排序，除非是一个非常小的数组或列表。
对于数组，使用java.util.Arrays.sort，是一个改进过的快速排序； 它不需要额外的内存，但是它不是稳定的(不能保证相等对象的顺序)。
对于ArrayList和LinkedList,他们都实现了接口java.util.List,可以使用 java.util.Collections.sort来排序，它是稳定的(保证相等对象的顺序)和平滑的(对于接近排好序的列表接近线性时间)， 但是它使用了额外的内存。
