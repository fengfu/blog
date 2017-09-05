---
title: 为什么Synchronized不能加在String和Integer等基本包装类型上
date: 2017-09-05 10:52:54
tags: Java
---

同步块不应该加在String或基本包装类型上，如Byte、Short、Integer、Long、Float、Double、Boolean、Character。

String不能用作同步块的参数是因为String为不可变对象，任何String对象的改变都将产生一个新的String对象，这也将导致前面加的锁不会被释放。

Integer、Boolean、Double、Long不能作为同步块参数的原因是他们是基本包装类型，包装类型有特殊的逻辑，用一句话说就是Java的自动封箱和解箱操作会导致这些对象在经过运算后不再是原来的对象。

用复杂的话说就是：当把基本变量赋值给包装类型的变量（其实编译过后的操作就是调用包装类型的静态方法valueOf）或者调用静态valueOf方法时：

* Boolean返回的是缓存的对象。
* 整型（Byte,Short,Integer,Long）会检查该数字是否在1个字节可表示的有符号整数范围内（-128~127），是则返回缓存对象，否则返回新对象。
* Character会缓存整型值为0~127的字符，同样会检查字符是否落在缓存范围中，是则返回，否则返回新对象。
* Double和Float的valueOf方法始终返回新对象。
