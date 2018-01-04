---
title: StringBuilder你应该知道的几件事情
date: 2018-01-02 20:33:21
tags: Java
---

字符串拼接是我们在编写程序时经常要写的代码，在很多的系统中，字符串相关的处理甚至占用了系统非常多的资源，尤其是内存。因此，在一些高性能的场景中，字符串拼接使用方式不合理，往往会导致无谓的内存开销，增加GC压力。

有些同学会直接使用"+"的方式进行字符串拼接，在Jdk7u40之前，这种方式我们是不推荐使用的，因为String是不可变对象，每次对String的"+"操作都会产生一个新的String对象，尤其是在对多个String进行拼接处理的时候，从而导致内存的浪费。但在Jdk7u40之后，-XX:+OptimizeStringConcat被默认打开，这样Jdk在进行Jit编译的时候，会自动将"+"形式的字符串拼接优化成StringBuilder append的方式。

但是，让编译器帮你优化这种方式你就可以高枕无忧了吗？答案是否定的，请看下面的内容。

## 1.new StringBuilder()合理吗？ ##
很多时候我们使用new StringBuilder()来初始化实例，但这种方式是否合理呢？

通过阅读StringBuilder的源码我们可以看到，StringBuilder的内部存储是使用char[]来实现的，char[]的初始容量是16。在进行append时，首先会检查char[]容量是否够用，如果不够，则会创建一个新的char[]，容量为原来容量的2倍，原来的char[]则会丢弃。看到这里，细心的同学可能会想了，这不是浪费了吗？是的，这就是不设置初始容量所带来的弊端，会导致一定的内存浪费。举个例子：

```
String a1 = "1234567890";//length 10
String a2 = "0123456789";//length 10
String a3 = "123456789";//length 9
String a4 = "987654321";//length 9

String a5 = new StringBuilder().append(a1).append(a2).append(a3).append(a4).toString();
```
上面这段代码的执行共创建了几个char[]呢？答案是4个，详细过程看下面的描述。
1. new StringBuilder()的时候，如果我们不指定StringBuilder的容量，那么会按照默认值是16创建第一个数组，我们命名为C1。在append(a1)的时候，容量是够用的；

2. 在append(a2)的时候，C1容量不够了需要扩容，于是创建一个容量为以前2倍的数组C2，并将旧的数组C1的内容以及a2的内容按顺序copy到新数组C2中，这时候C2的大小是32；

3. 在append(a3)的时候，C2的容量是够用的，所以直接将a3的内容copy进C2；

4. 在append(a4)的时候，C2的容量也不够用了，于是再创建一个容量为C2两倍的数组C3，并将旧的数组C2的内容以及a4的内容按顺序copy到新数组C3中，这时候C3的大小是64；

5. 在toString的时候，是调用的new String(value, 0, count)。这样并不是将C3的内容直接转换成String，而是会copy出一个副本C4再根据C4创建出一个String来；这一点，建议大家仔细去看看StringBuilder的toString代码，你会发现更多的真相……

![](https://farm5.staticflickr.com/4690/39073757202_23dcf8ee35_b.jpg)

看完上面的描述，你会觉得这个过程中浪费了多少内存呢？最起码C1、C2是没用了的，也就是说浪费掉了48byte的空间。那如果你最初根据预估的字符串大小，设置了StringBuilder初始容量为38，那么就不会存在这样的浪费了，同时也会省去StringBuilder扩容的时间。

所以，在使用StringBuilder的时候，还是估算一下字符串大小，乖乖的设置一个初始容量吧。

## 2.不要重复创建StringBuilder ##
这里说的重复是指在循环中重复new StringBuilder实例。前面我们也说过，StringBuilder的扩容和toString()方法会造成大量的char[]浪费，如果你在一个循环里面创建了很多StringBuilder实例，那么可以想象浪费的内存有多大……所以在一个循环里面复用StringBuilder比较好的方式是调用StringBuilder.setLength(0)重置指针进行复用，这样最起码就能避免再次扩容的内存消耗了。

## 3.StringBuilder.toString()造成的内存浪费 ##
前面我们也提到过，每次StringBuilder.toString的时候，会copy一个char[]生成一个String对象，这样就相当于白白浪费了StringBuilder里面的数组。当然这里可以采用一些黑科技来避免这种浪费，比如用Unsafe这种黑科技，绕过构造函数直接给String的char[]属性赋值，示例代码如下：
```
  Unsafe unsafe =
  long fieldOffset = Unsafe.getUnsafe();
  unsafe.objectFieldOffset(String.class.getDeclaredField("value"));
  String ret = new String();

  unsafe.putObjectVolatile(ret, fieldOffset, char[]);//char[]为StringBuilder中的char[]
```
这种方式虽然能够避免内存浪费，但也存在一定的风险，所以使用的时候还是要慎重。

## 4.StringBuilder vs StringBuffer ##
StringBuffer与StringBuilder都是继承于AbstractStringBuilder，唯一的区别就是StringBuffer的函数上都有synchronized关键字。
