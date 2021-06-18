# Large-scale Python (2)

上篇谈了 [Python 的项目构建][Series1]，这次讲讲 Python 的代码编写。

# Python 的代码编写

> The reason we say programming remains an art, not a science or an engineering discipline, is because we haven’t as yet been able to break it down into component steps and mechanize it.
- [Stanley B. Lippman - Is Programming an Art?](http://msdn.microsoft.com/en-us/magazine/cc163546.aspx)

Python 以易学易用闻名，但想写出 [Pythonic](http://programmers.stackexchange.com/questions/119913/how-can-i-learn-to-effectively-write-pythonic-code) 的代码着实不易。但这些都不是重点。一旦项目上了规模，就是需要一种扼杀码农天赋，把他们都变成搬砖工人的工业化语言。这也是为何 Java 在企业级开发中大行其道的原因之一。

## 基础库

处于所有业务代码的最底层，不为人知却最有可能被称为艺术的，就是这些服务大众的基础库。

同其它语言一样，Python 基础库的设计原则也是“[抽象](http://en.wikipedia.org/wiki/Abstraction_(computer_science))”。由于见识不足，我们的基础库在第一天并没有这样做，而是定位成“工具集”，为以后的演化带来不小的麻烦。

“[类型系统](http://en.wikipedia.org/wiki/Type_system)”是抽象化的基石。对于严谨自洽的基础库，给所有类、函数定义都加上类型约束，不失为一个好的实践：

{% highlight python linenos %}
from typing import List


class Node:

    def __init__(self):
        pass

    def nodename(self) -> str:
        pass

    def set_nodename(self, name: str) -> None:
        pass


class NodeGroup:

    def __init__(self):
        pass

    def nodes(self) -> List[Node]:
        pass


def business_func(nodenames: List[str]):
    pass


ng = NodeGroup()
business_func(ng.nodes())
{% endhighlight %}

这段代码用 [mypy][mypy] 处理会输出：

    base_libs.py, line 30: Argument 1 to "business_func" has incompatible
    type List[Node]; expected List[str]

看到了吗，熟悉的 Java 又回来了。[Mypy][mypy] 是 Python 之父 Guido [钦点](https://mail.python.org/pipermail/python-ideas/2014-August/028618.html)的类型检查工具。

## 再来点封装

基础库的接口设计一般遵循[最小化原则](http://en.wikipedia.org/wiki/Interface_segregation_principle)，强调的是一个接口拥有的行为应该尽可能的小。实用中，偷懒的程序员总是需要在基础库之上封装一些便捷方法。这类封装可以适当放宽甚至去掉类型约束，充分发挥动态类型的优势。

## 最终用户态

到了实现业务逻辑，或者写一些一次性的小工具这一层，就可以天马行空，怎么舒服怎么写。毕竟 Python 还是一种脚本语言。

## 设计模式，回调和函数式

最近一组编程语言的[漫画](http://blog.jobbole.com/77608/)，说设计模式、回调分别是 Java、Javascript 的高阶技能。[函数式][FP]在 Ruby 里用的比较多。实际中，我们发现任何高阶技能都是大忌。为了项目的可持续维护性考虑，不建议在非必要的情况下，盲目炫技。

例如，稍微上点档次的设计模式 [Visitor](http://en.wikipedia.org/wiki/Visitor_pattern)，每次不重温一遍就不知道怎么写，想自觉的用起来就更加不可能了。

[KISS](http://en.wikipedia.org/wiki/KISS_principle) 原则暗自契合 Python 的真意，如果运用得当，将 Python 打造成 Java 那样的民工语言也不是不可能。

本篇到此结束。请看下篇 [Python 的运行时][Series3]。


[mypy]: http://www.mypy-lang.org
[FP]: http://en.wikipedia.org/wiki/Functional_programming
[Series1]: {% post_url 2014-12-29-large-scale-python-1 %}
[Series2]: {% post_url 2014-12-30-large-scale-python-2 %}
[Series3]: {% post_url 2014-12-31-large-scale-python-3 %}

