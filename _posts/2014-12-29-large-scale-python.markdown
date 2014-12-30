---
layout: post
title:  "Large-scale Python"
date:   2014-12-29 22:00:00
categories: jekyll update
---

这一年百分之七十的时间都在做 Python 相关的工作。Python 语言的轻便、灵巧，让码工时间充满快感，创意连连。而 Java 却是啰嗦，极其的啰嗦。玩了一段时间 Django [^Django] 之后，越发的喜爱上了这门语言。

本文拟从以下几个方面来探讨 Python [^CPy] 在 Large-scale [^LS] 项目中的一些应用要点，内容都是满满的原创经验：

* Python 的项目构建
    - Maven 吗？
    - 语言风格
    - IDE
    - 测试
* Python 的代码编写
    - 基础库
    - 再来点封装
    - 最终用户态
    - 设计模式，回调和函数式
* Python 的运行时
    - 性能
    - GC
    - C10k
    - 优化
* Python 的软件发布
    - Virtualenv

- - - 

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

[^RDA]: 能直接安装至目标系统的服务配置包。
  
## 解决模块重用

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

在不同的构建环境(例如 UserHome 或者 Jenkins-CI Slave)中如何让 project-1 和 project-2 顺利发现 common-libs 并依赖[^Dep]之完成构建？这个问题在 Java 世界里很好解决：假如 common-libs 的产出是 jar 包。

    cd common-libs
    mvn install (or mvn deploy)
    cd ../project-1
    mvn deploy
    cd ../project-2
    mvn deploy

在不搭建内部 [PyPI][PyPI] 的前提下，这个问题一直没有找到一个完美的解决方案。实际操作中，是让 `cd common-libs; mvn install` 调用至 `python setup.py install --user`，在本地安装 common-libs。

[PyPI]: http://en.wikipedia.org/wiki/Python_Package_Index
[^Dep]: 事实上 Python 的 Project 构建并没有强依赖关系，不像 Java 缺少必要的 jar 则无法成功编译。但是，在构建的时候使用 Pylint 工具进行伪编译，是控制 Large-scale 项目代码质量的最佳实践。

## Maven 吗？

- - -

# 结论 


https://github.com/aclisp/archived/tree/master/scratch-01/mvn-python-packaging

- - -

[^Django]: Django 大约是 Python 世界中的 Spring Framework。
[^LS]: 又名企业级。特点是开发人员多，业务量大，接口多样，架构不定复杂。
[^CPy]: 以下出现的 Python 字样特指 [CPython](http://en.wikipedia.org/wiki/CPython)。
