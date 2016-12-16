---
title: Java的SPI机制
date: 2015-02-21 19:35:09
tags: Java
---

## 什么是SPI ##
SPI是Service Provider Interface（服务提供者接口）的缩写，从字面意思也能看出，SPI是面向服务提供者的。也就是说，有人定义了接口，那么围绕该接口提供服务的第三方则针对接口提供自己的服务。说到这里，最典型的应该是我们常用的日志框架，比如common-logging、jdbc Driver。
## SPI的优点 ##
基于SPI机制，我们可以很方便地构建出易于扩展的应用。我们使用的很多应用框架，比如dubbo，其中的extension就是基于SPI构建的。

## Java SPI规范 ##
1. 在META-INF/services/目录下创建以接口全名命名的文件,该文件内容为接口具体实现类的全名;如我有一个实现类io.fengfu.learning.spi.HelloWorldServiceBB8Impl，实现了io.fengfu.learning.spi.HelloWorldService接口，那么就需要在META-INF/services目录下创建一个名称为io.fengfu.learning.spi.HelloWorldService的文件，文件的内容为io.fengfu.learning.spi.HelloWorldServiceBB8Impl。如果我有很多个实现类，那么只需要将实现类全名按行编写即可;
2. 接口具体实现类必须有一个不带参数的构造方法;
3. 使用ServiceLoader类动态加载META-INF中的实现类;
4. 如SPI的实现类为Jar则需要放在主程序classPath中;

## SPI示例 ##

1,定义一个接口

    package io.fengfu.learning.spi;

    /**
     * Created by fengfu.qu on 2014/2/4.
     */
    public interface HelloService {
        public String sayHello(String name);
    }

2, 实现之：

    package io.fengfu.learning.spi;

    /**
     * Created by fengfu.qu on 2014/2/4.
     */
    public class HelloServiceC3poImpl implements HelloService {
        public String sayHello(String name) {
            return "Hello, " + name + ", I'm C3po.";
        }
    }

	package io.fengfu.learning.spi;

    /**
     * Created by fengfu.qu on 2014/2/4.
     */
    public class HelloServiceR2D2Impl implements HelloService {
        public String sayHello(String name) {
            return "Hello, " + name + ", I'm R2D2.";
        }
    }

	package io.fengfu.learning.spi;

    /**
     * Created by fengfu.qu on 2014/2/4.
     */
    public class HelloServiceBB8Impl implements HelloService {
        public String sayHello(String name) {
            return "Hello, " + name + ", I'm BB8.";
        }
    }

3, 在META-INF/services目录下创建io.fengfu.learning.spi.HelloService文件，在文件中添加以下内容:

	io.fengfu.learning.spi.HelloServiceC3poImpl
    io.fengfu.learning.spi.HelloServiceR2D2Impl
    io.fengfu.learning.spi.HelloServiceBB8Impl

4, 创建测试类：

    package io.fengfu.learning.spi;

    import java.util.Iterator;
    import java.util.ServiceLoader;

    /**
     * Created by fengfu.qu on 2014/2/4.
     */
    public class SPITest {
        public static void main(String[] args) {
            ServiceLoader<HelloService> loader = ServiceLoader.load(HelloService.class);
            Iterator<HelloService> it = loader.iterator();
            while(it.hasNext()){
                HelloService helloSPI = it.next();
                System.out.println(helloSPI.sayHello("Fengfu"));
            }
        }
    }

5, 运行，结果为：
    
	Hello, Fengfu, I'm C3po.
    Hello, Fengfu, I'm R2D2.
    Hello, Fengfu, I'm BB8.

由此看出，3个是实现类都被运行了。
