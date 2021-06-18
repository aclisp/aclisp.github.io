# Large-scale Python (3)

上篇谈了 [Python 的代码编写][Series2]，这次讲讲 Python 的运行时。

# Python 的运行时

## 性能

尽管 Python 反对 [premature optimization](http://en.wikipedia.org/wiki/Program_optimization#When_to_optimize)，作为一个有资历的 C++ 程序员，每学习一门新语言，我还是喜欢把它跟 C++ [比一比](http://benchmarksgame.alioth.debian.org)。 结果显示Python 确实慢，跟 Java、Go 之流相比慢的发指，甚至不如 Ruby。

基本提速手段有下列这些：

* 换用支持 [JIT](http://en.wikipedia.org/wiki/Just-in-time_compilation) 的 [PyPy](http://pypy.org) 虚拟机。
* 将[计算密集型](http://en.wikipedia.org/wiki/CPU-bound)模块用 C++ 编写，可以利用 [Boost.Python](www.boost.org/doc/libs/release/libs/python/)。
* [IO密集型](http://en.wikipedia.org/wiki/I/O_bound)任务，考虑用[异步框架](https://www.paypal-engineering.com/2014/12/10/10-myths-of-enterprise-python/#python-lacks-concurrency)实现。

## GC

Python 使用基于引用计数的 GC。官方 FAQ 里有一些[思考](https://docs.python.org/3/faq/design.html#how-does-python-manage-memory)。由于我们的实际应用主要是做软件部署和服务配置，有关 GC 在大规模吞吐量下的表现还有待研究。

## C10k

[C10k](http://en.wikipedia.org/wiki/C10k_problem) 是处理大量连接数的问题。Java 有 [Netty](http://en.wikipedia.org/wiki/Netty_%28software%29)，Python 也不差，有 [Twisted](http://en.wikipedia.org/wiki/Twisted_(software))。说白了，就是对几种操作系统原语[^select]的封装，进而实现了以“[反应堆](http://en.wikipedia.org/wiki/Reactor_pattern)”模式为首的一众网络设计模式。

## 优化

提高 Python 代码运行效率的诀窍无他，在于更深入的研究和长时间的磨练：

* 理解 Python 的对象模型[^e1][^e2]。话说，让我的 C++ 技能登堂入室的一本书[^e3]也是讲对象模型的。学语言的历程还真是异曲同工。
* 如果你跟我一样是半路出家，跟其它语言多多横向比较[^e4]
* 读一本好书。例如这本书，光看看[目录列表](http://www.effectivepython.com)就会有不少收获。

# 结束语

2014 年最后一天，带着一年来的感悟，完成了这一系列三篇文章。对 Python 的认识，从最初偶尔用用，到现在深度思考。其间各种资料收集不易，各种真知都来自实践，记在这里备查。

---
[^select]: Nginx 的文档里有个[详细说明](http://nginx.org/en/docs/events.html)。
[^e1]: ["How to think like a Pythonista"](http://python.net/crew/mwh/hacks/objectthink.html)
[^e2]: [Goodger, David. "Code Like a Pythonista: Idiomatic Python"](http://python.net/~goodger/projects/pycon/2007/idiomatic/handout.html)
[^e3]: [Stanley B. Lippman. _Inside the C++ Object Model_](http://book.douban.com/subject/1484262/)
[^e4]: ["Python Is Not Java"](http://dirtsimple.org/2004/12/python-is-not-java.html)

[Series1]: {% post_url 2014-12-29-large-scale-python-1 %}
[Series2]: {% post_url 2014-12-30-large-scale-python-2 %}
[Series3]: {% post_url 2014-12-31-large-scale-python-3 %}
