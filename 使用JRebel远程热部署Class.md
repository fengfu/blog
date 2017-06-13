---
title: 使用 jrebel 远程部署开发环境的 dubbo 服务
date: 2017-04-11 21:08:09
tags: Java
---


本文转自http://wulfric.me/2017/04/jrebel/ ，如果转载请尊重原作者版权。


> 声明: 本文采用 [CC BY-NC-ND 4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/deed.zh "姓名标示-非商业性-禁止改作4.0国际") 授权。


<figure><a class="post-image" rel="post-image" href="">![jrebel ](https://zeroturnaround.com/wp-content/uploads/2011/02/jrebel-dl.png "jrebel")</a>
</figure>

在开发 java 程序的时候，如果改动非常频繁，每次改动都要重新打包、部署，对于很重的应用（比如本文所说的 dubbo provider 服务），这样反复的流程十分消耗精力和热情。我们需要一个工具，当代码作改动之后能够立即看到效果。Java 在 1.4 的时候引入了 HotSwap 技术，允许调试者使用同一个类标识来更新类的字节码。这意味着所有对象都可以引用一个更新后的类，并在它们的方法被调用的时候执行新的代码，这就避免了无论何时只要有类的字节码被修改就要重载容器的这种要求<sup id="fnref:hotswap-jrebel">[1](#fn:hotswap-jrebel)</sup>。不幸的是，这种重定义仅限于修改方法体—除了方法体之外，它既不能添加方法或域，也不能修改其他任何东西。

和 PHP 这样的脚本语言不一样，Java 部署是一个令人心烦的问题。一个运行中的 Java 程序相当于一辆在公路上奔跑的汽车，而 HotSwap 技术只能让你更换坐垫，但是让你想要更换轮胎的时候，HotSwap 就帮不上忙了，这个时候你只能停下来，换掉轮胎，再重新启动。而对于 PHP 来说，每个请求都会新开一个进程加载所有的 .php 文件编译和执行，所以不存在这个问题。运行 PHP 程序相当于给静止的汽车拍照，你大可以换完轮胎之后再拍照，这没有任何不良影响。（这个比喻可能不太恰当，却很有趣：在比喻中，静态的语言看起来是运动的，而动态语言看起来是静止的）

[JRebel](https://zeroturnaround.com/software/jrebel/) 是 java 的热部署插件，它监控已编译的 .class 文件，只要有变动，就立即更新在部署好的应用上，能够实现实时查看代码变化的功能。

<figure><a class="post-image" rel="post-image" href="">![skip build, package and deploy](https://zeroturnaround.com/wp-content/uploads/2016/11/JR_devcycle_2016_c.png "skip build, package and deploy")</a>

<figcaption>skip build, package and deploy</figcaption>

</figure>

HotSwap 是工作在虚拟机层面上，且依赖于 JVM 的内部运作，JRebel 用到了 JVM 的两个显著的功能特征—抽象的字节码和类加载器。类加载器允许 JRebel 辨别出类被加载的时刻，然后实时地翻译字节码，用以在虚拟机和可执行代码之间创建另一个抽象层<sup id="fnref:hotswap-jrebel:1">[1](#fn:hotswap-jrebel)</sup>。这种技术也应用在 zeroturnaround 家其他的工具，比如 xrebel。

如果继续使用上面的比喻，JRebel 相当于在汽车运行过程中，先额外生成一个轮胎，然后以迅雷之势替换掉目标轮胎—汽车不需要停止装载和再启动。

## 安装和使用

JRebel 的安装参照官网即可，我是直接在 Intellij 的插件中心安装的，速度比较慢，插件也比较大，挂了代理才下载完成。下载完毕重启 IDE 即可使用。

JRebel 是收费插件，第一次打开的时候需要激活，网上有各种神奇教程可以参考。其实这个插件官方提供了免费的激活码，只要需要你注册一个[帐号](https://my.jrebel.com/)，并关联一个 twitter 或者 facebook 帐号即可。jrebel 需要这个关联帐号的写权限，会定期发送使用统计到你的 timeline，你可以使用一个小号来关联。

安装和激活完毕之后，就可以在 IDE 的配置页看到 JRebel 选项了。

<figure>[![jrebel preference](http://wulfric.qiniudn.com/java/jrebel-preference.png-article.webp "jrebel preference")](http://wulfric.qiniudn.com/java/jrebel-preference.png-origin.webp)

<figcaption>jrebel preference</figcaption>

</figure>

## 运行

为了能够比较熟悉该插件的使用，建议按照官方教程，先通过 IDE 本地启动，再通过命令行启动，然后尝试部署到远端服务器。

当在 console 窗口观察到这些输出之后，说明应用正确启用了 JRebel 插件。

    2017-04-01 16:27:31 JRebel:  #############################################################
    2017-04-01 16:27:31 JRebel:
    2017-04-01 16:27:31 JRebel:  JRebel Agent 7.0.6 (201703201213)
    2017-04-01 16:27:31 JRebel:  (c) Copyright ZeroTurnaround AS, Estonia, Tartu.
    2017-04-01 16:27:31 JRebel:
    2017-04-01 16:27:31 JRebel:  Over the last 2 days JRebel prevented
    2017-04-01 16:27:31 JRebel:  at least 0 redeploys/restarts saving you about 0 hours.
    2017-04-01 16:27:31 JRebel:
    2017-04-01 16:27:31 JRebel:  JRebel started in remote server mode.
    2017-04-01 16:27:31 JRebel:
    2017-04-01 16:27:31 JRebel:
    2017-04-01 16:27:31 JRebel:  #############################################################

### IDE 运行

IDE 运行本地应用最简单。参见[官方教程](https://zeroturnaround.com/software/jrebel/quickstart/intellij/?run=ide#!/server-configuration)。

1.  执行<span class="codespan">mvn clean</span>清空 target 目录；
2.  打开 「View > Tool Windows > JRebel」调出 JRebel Panel 编辑窗，点击「generate rebel.xml」按钮选中需要监控和热部署的模块。因为是本地执行，所以另外一个「generate rebel-remote.xml」 不需要选中；
3.  将以前通过 「Run 'MyApp'」或者「Debug 'MyApp'」来运行的命令改成「Run with JRebel 'MyApp'」或者「Debug with JRebel 'MyApp'」即可；
4.  修改代码，然后在「Build」菜单下选择合适的构建选项（只构建单个文件/构建模块/构建项目）；
5.  访问接口，查看结果是否实时改变。

### 命令行运行

在 JRebel 设置页的「Startup」标签中选择「Run locally from command line」， 选择合适的 jvm 和目标环境。我的 dubbo 使用了 netty 部署，所以选择了「Standalone application」，给出的启动参数如下。

    # agentpath 是必需的，foo.bar.MyApp 是你要运行的程序
    java -agentpath:/Users/xxxx/Library/Application Support/IntelliJIdea2017.1/jr-ide-idea/lib/jrebel6/lib/libjrebel64.dylib foo.bar.MyApp

P.S.: 注意上面的「Application Support」需要换成「Application\ Support」。

dubbo 一般推荐使用 start.sh 脚本来启动应用，我们编辑 start.sh 文件，在启动的具体位置那里，将「-agentpath:…」添加在 java 后面，大致如下。

    java -agentpath:... $JAVA_OPTS $JAVA_MEM_OPTS $JAVA_DEBUG_OPTS -classpath ...

各种目标环境的配置方法有着或大或小的区别，请按照自己的目标环境跟着 JRebel 给的配置方案来设置。

1.  配置启动参数，如上所述；
2.  选择需要热部署的模块；
3.  执行<span class="codespan">start.sh</span>部署应用；
4.  Run with JRebel 'MyApp'；
5.  修改代码，构建改动的文件；
6.  访问接口，查看结果是否实时改变。

从输出结果可以看到，JRebel 接管了 console。

### 远程服务器运行

官方[教程1](https://zeroturnaround.com/software/jrebel/quickstart/intellij/?run=remote#!/server-configuration)，[教程2](https://manuals.zeroturnaround.com/jrebel/remoteserver/intellij.html)更好<sup id="fnref:jrebel-doc">[2](#fn:jrebel-doc)</sup>。

同上，在 JRebel 配置页选择「Run on a remote server」，根据给出的建议配置好启动项。

    java -agentpath:... -Drebel.remoting_plugin=true ... -jar ...

在 JRebel 配置页添加远程服务器地址，注意要是能够 http 访问的 url 才可以。

<figure><a class="post-image" rel="post-image" href="">![添加远程服务器](https://manuals.zeroturnaround.com/jrebel/_images/intellij-add-remote-server.png "添加远程服务器")</a>
</figure>

添加远程服务器

1.  配置启动参数和添加远程服务器，如上所述；
2.  选择需要热部署的模块，注意这个时候「generate rebel.xml」和「generate rebel-remote.xml」 两个都要选中；
3.  将包含 rebel.xml 和 rebel-remote.xml 的代码部署到远程服务器。本地打包并上传，或者代码上传并在远程服务器打包都可以；
4.  在远程服务器上<span class="codespan">start.sh</span>启动应用；
5.  修改代码，构建改动的文件；
6.  访问接口，查看结果是否实时改变。

P.S.: 教程1似乎有些问题，远端启动应用后，本地不需要再「Run with JRebel 'MyApp'」了。

P.S.: 在官方教程里，「Run with JRebel 'MyApp'」和「generate rebel.xml」checkbox 的图标一模一样，注意区分：有的是让你启动，有的是选中。

P.S.: JRebel 对 Spring 的 bean 注入默认就有很好的支持，参见 [JRebel Spring](https://zeroturnaround.com/software/jrebel/learn/jrebel-spring-integration/)。它可以根据 xml 文件或者注解自动重新装载 bean。还支持动态实时地在 Spring 中添加 bean 和依赖。

1.  HotSwap 和 JRebel 原理[原文](https://dzone.com/articles/reloading-java-classes-401)，[译文](http://www.hollischuang.com/archives/598) [↩](#fnref:hotswap-jrebel) [↩<sup>2</sup>](#fnref:hotswap-jrebel:1)

2.  参照官方的[详细使用文档](https://manuals.zeroturnaround.com/jrebel/index.html)是更好的选择 [↩](#fnref:jrebel-doc)
