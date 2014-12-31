---
layout: post
title:  "Large-scale Python (1)"
date:   2014-12-29
categories: jekyll update
---

这一年百分之七十的时间都在做 Python 相关的工作。Python 语言的轻便、灵巧，让码工时间充满快感，创意连连。而 Java 却是啰嗦，极其的啰嗦。玩了一段时间 Django[^Django] 之后，越发的喜爱上了这门语言。

本文拟从以下几个方面来探讨 Python[^CPy] 在 Large-scale[^LS] 项目中的一些 __应用和实践__，内容都是满满的原创经验：

* Python 的项目构建
* [Python 的代码编写][Series2]
* [Python 的运行时][Series3]

# Python 的项目构建

新项目的构建要解决的问题如下：

## 标准化目录结构

一个标准的目录结构有助于项目组成员互相理解，减少沟通成本。由于继承性的需要，实际采用的是 [maven 模式](http://maven.apache.org/guides/introduction/introduction-to-the-standard-directory-layout.html)。于是一个完整的目录结构是这个样子的：

    project-root
    ├── pom.xml
    └── src
        ├── main
        │   ├── config        
        │   ├── python        (注1)
        │   ├── rpm-resources (注2)
        │   └── scripts       (注3)
        └── test
            ├── config
            ├── features      (注4)
            └── python        (注5)

注：

1. 存放 Python 的 [Package](https://docs.python.org/3/tutorial/modules.html#packages)
2. 存放需打包的同等目录结构
3. 存放 [Scriptlet](http://mojo.codehaus.org/rpm-maven-plugin/adv-params.html#Scripts)
4. 存放 [Lettuce Feature 文件](http://lettuce.it/tutorial/simple.html#tutorial-simple)
5. 存放 Python 的 UT 和 FT

它与 GitHub 上标准的 Python 项目（如 [Django](https://github.com/django/django)）不同。后者是纯粹的 Python 包，用于发布 [PyPI][PyPI]。而企业闭门开发，需要利用内部的 maven 仓库；同时这里[^RDA]更需要一种混合包，以 Python 实现为主体，有限度集成 Shell 脚本。

## 模块重用

经过一段时间的演化，project-1 和 project-2 的公用部分被独立出来成为新的 common-libs 

    git-repo-root 
    ├── common-libs
    ├── project-1
    └── project-2

或者项目代码跨越了 git 库

    git-repo-1 
    └── common-libs

    git-repo-2
    ├── project-1
    └── project-2

在不同的构建环境(例如 UserHome 或者 Jenkins-CI Slave)中如何让 project-1 和 project-2 顺利发现 common-libs 并依赖[^Dep]之完成构建？这个问题在 Java 世界里很好解决：假如 common-libs 的产出是 jar 包，

    cd common-libs
    mvn install (or mvn deploy)
    cd ../project-1
    mvn deploy
    cd ../project-2
    mvn deploy

在不搭建内部 [PyPI][PyPI] 的前提下，这个问题一直没有找到一个完美的解决方案。实际操作中，是让 `cd common-libs; mvn install` 调用至 `python setup.py install --user`，在本地安装 common-libs。

## 语言风格

程序员都是个性动物，写出的代码各种风格都有。在大型项目中，统一的代码风格非常有必要。Python 哲学里也有 preferably only one way to do it [^Zen]。流行的 [Lint](http://en.wikipedia.org/wiki/Lint_%28software%29) 工具有这么几种：

* Syntax errors and inconsistencies (using [Pyflakes](https://launchpad.net/pyflakes) or [Pylint](http://www.pylint.org/))
* [PEP8](https://www.python.org/dev/peps/pep-0008/) violations 
* [PEP257](https://www.python.org/dev/peps/pep-0257/) violations

这里边，轻量级的是`PEP8`和`Pyflakes`。建议任何时候都开。当你灵感满溢思如泉涌啪啪啪敲键盘时，它们在后台默默的保持最基本检查，绝不干扰思路，充分体现自由。重量级的`Pylint`和`PEP257`可以作为持续集成任务定时对整个代码库检查。当然，如果你是追求完美的处女座，全部打开也没有问题的，妈妈再也不用担心我的代码写的乱七八糟了。

实践中，我们把`Pylint`集成到`maven compile`，使提交入 git 的代码都是 lint 过的。

## IDE

工欲善其事，必先利其器。好的 IDE 能帮助做这些事情：

* 生成遵循标准化目录结构的项目模板
* 提供语言风格的检查开关
* 管理 Python 运行环境
* 语法高亮
* 智能提示
* 自动完成

土豪就上公认神器 [PyCharm](https://www.jetbrains.com/pycharm/) 专业版。否则，Eclipse 装上 [PyDev](http://pydev.org/) 也能凑合。极客就用 Sublime + [Anaconda](http://damnwidget.github.io/anaconda/)，我是用的很高兴的。

本篇到此结束。请看下篇 [Python 的代码编写][Series2]。

---
[^Django]: Django 大约是 Python 世界中的 Spring Framework。
[^LS]: 又名企业级。特点是开发人员多，业务量大，接口多样，架构不定复杂。
[^CPy]: 以下出现的 Python 字样特指 [CPython](http://en.wikipedia.org/wiki/CPython)。
[^Zen]: [The Zen of Python](https://www.python.org/dev/peps/pep-0020/)
[^RDA]: 能直接安装至目标系统的服务配置包。
[^Dep]: 事实上 Python 的 Project 构建并没有强依赖关系，不像 Java 缺少必要的 jar 则无法成功编译。但是，在构建的时候使用 Pylint 工具进行伪编译，是控制 Large-scale 项目代码质量的最佳实践。

[PyPI]: http://en.wikipedia.org/wiki/Python_Package_Index
[Series1]: {% post_url 2014-12-29-large-scale-python-1 %}
[Series2]: {% post_url 2014-12-30-large-scale-python-2 %}
[Series3]: {% post_url 2014-12-31-large-scale-python-3 %}

