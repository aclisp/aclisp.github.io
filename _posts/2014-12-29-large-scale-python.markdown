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

从 Large-scale 项目的实际需求出发，

## Maven 吗？

- - -

# 结论 


https://github.com/aclisp/archived/tree/master/scratch-01/mvn-python-packaging

- - -

[^Django]: Django 就大约是 Python 世界中的 Spring Framework。
[^LS]: 又名企业级。特点是开发人员多，业务量大，接口多样，架构倒是不一定复杂。
[^CPy]: 写技术要文风严谨。以下出现的 Python 字样特指 [CPython](http://en.wikipedia.org/wiki/CPython)。