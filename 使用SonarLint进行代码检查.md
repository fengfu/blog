---
title: 使用SonarLint编写高质量代码
date: 2017-12-08 18:40:25
tags: Java
---

Sonarqube是一个功能非常强大的代码质量检查、管理的工具。能够识别多种常用的编程语言，并能够通过设置不同的Rule
Sonar是一个代码质量管理的开源工具，它通过插件的形式能够识别常见的多种编程语言（例如Java, C#, PHP, Pythod等）代码质量问题。Sonar可以帮你分析出以下代码质量问题：
1.不遵循代码标准；
2.潜在的缺陷，比如空指针、bug；
3.代码重复；
4.注释率不足或过高；
5.糟糕的复杂度，比如if/循环嵌套太多、类/方法太大；
6.缺乏单元测试；

在公司中，一般是把SonarQube部署在服务器端，当开发人员提交代码时，Jenkins触发SonarQube进行代码检查，开发人员根据代码检查结果进行问题修复。但是这样，开发人员往往只会关注被阻断的代码问题，对于Sonar提示的设计、性能方面的问题往往视而不见。
所幸现在SonarSource开发了SonarLint插件，可以在IDE(Intellij Idea、Eclipse)中嵌入，这样开发人员不仅能使用SonarLint中内置的代码检查规则进行代码检查，也可以连接到远端服务器拉取远端规则。有了它，我们就可以在编写代码的过程中根据SonarLint的提示编写高质量代码了。

下面，将以Intellij Idea为基础介绍SonarLint插件的使用。

## 1. 安装 ##

在Idea的Plugins界面搜索“SonarLint”，在检索结果列表页找到对应的SonarLint，点击右侧的“Install”按钮，安装完毕重启Idea后即可完成插件的安装。
![安装插件](https://farm5.staticflickr.com/4634/38195936814_d3318454b4_b.jpg)

## 2. 代码检查 ##

SonarLint插件安装后，就可以使用它对Idea中的项目进行代码检查了。SonarLint能够对单个文件、整个项目、从VCS（版本控制系统，比如git、svn等）拉取的被修改文件这3类进行检查。

开启SonarLint的检查有3个入口：

1. Analyze->SonarLint，如下图：
![](https://farm5.staticflickr.com/4558/38195610794_c63be62dd8_b.jpg)

2. 在打开的文件编辑区域点击右键，在弹出的上下文菜单中选择，如下图：

![](https://farm5.staticflickr.com/4562/38875276892_7c0f79c2a4_z.jpg)

此方式只能对当前文件进行检查。

3. 编辑区底部的Panel，这里可以在"Current file" Tab区域对当前文件进行检查，也可以在“Project files”Tab区域对文件进行全量和从VCS（版本控制系统，比如git、svn等）拉取的被修改文件进行检查，如下图：

![](https://farm5.staticflickr.com/4519/38196120834_fa47b16e02_b.jpg)

执行代码检查后，在下面的SonarLint Panel区域就能显示代码检查的结果了：

![](https://farm5.staticflickr.com/4566/38024453305_b355b33948_z.jpg)

## 3. 远程配置 ##

前面提到的代码检查，默认使用SonarLint内置的规则。我们也可以连接远端的SonarQube服务器，拉取服务器上配置的规则对代码进行检查，这样就能与代码提交时触发的检查规则保持一致了，具体方式如下：
1. 首先在下图的界面点击“+”创建新的SonarQube Server：
![](https://farm5.staticflickr.com/4557/38024453645_871bb2cd83.jpg)
2. 在New SonarQube Server界面中选择右侧的选项，并输入SonarQube URL，点击next：
![](https://farm5.staticflickr.com/4557/27134414809_a16b3fc406_b.jpg)
3. 在Authentication界面选择Login/Password，并输入用户名和密码：
![](https://farm5.staticflickr.com/4532/27134414729_f52684145d_b.jpg)
验证通过后点击Finish即可完成SonarQube Server的配置。
4. 在SonarLint界面选择Server中配置的project，要与当前项目一致，否则可能会拉取错误的规则配置。
![](https://farm5.staticflickr.com/4739/38875276722_f5b65063f0_b.jpg)

经过上面4步，你就可以通过SonarLint拉取的服务器上的代码规则进行代码检查了。Have fun:)
