# 函数式编程的思考

如今的互联网时代，[函数式编程](http://en.wikipedia.org/wiki/Functional_programming)越来越频繁的进入普通程序员的眼球。这个当年仅仅存在于学术界的玩具，渐渐影响到工业界。到底它是不是从业人员的救星，能够产出高质量代码？在团队开发中，它是否具有可行性、可维护性？我只从一个微小细节，来记录对它的一些感受。

先从一段代码看起：

> 在函数式编程中，我们不应该用循环迭代的方式，我们应该用更为高级的方法，
> 如下所示的Python代码
>
>     name_len = map(len, ["hao", "chen", "coolshell"])
>     print name_len
>     # 输出 [3, 4, 9]
>
> 你可以看到这样的代码很易读，因为，__这样的代码是在描述要干什么，而不是怎么干。__
- [酷壳：函数式编程](http://coolshell.cn/articles/10822.html)

我们可以这样理解这段代码：

* 它意图取得一个字符串序列每个元素的长度
    - 输入：一个序列，每个元素是一个字符串
    - 输出：一个序列，每个元素是对应的字符串长度
* 它写的很简洁！
* 为了理解它，我们在大脑里进行了如下思维活动，用 Python 代码表述出来就是

        aStrList = ["hao", "chen", "coolshell"]
        aIntList = []
        for i in range(len(aStrList)):
            aIntList.append( len(aStrList[i]) )

如果真是这么理解，可就**落了下乘**：我们仍然没有逃出[命令式编程](http://en.wikipedia.org/wiki/Imperative_programming)的牢笼，不知不觉中用深受历来编程教育荼毒的思维模式[^1]，完成了一次“怎么干”的思考。而函数式编程的精髓是描述“干什么”。

看 Python 里 `map` 的帮助文档：

>    map(function, sequence[, sequence, ...]) -> list
>
>    Return a list of the results of applying the function to the items of
>    the argument sequence(s).

你看，不会提是不是顺序遍历，是不是 append。`map` 抽象了 `for each ... append to result` 的效果，而不是过程。这也是过程式编程的缺点所在：有太多副作用（ side effect ），不清楚其中哪些才是需要的“效果”，哪些只是“过程”的副产物。

`map` 描述的**仅仅**是这样一个转换：

    map(f, [a1, a2, a3...]) => [f(a1), f(a2), f(a3)...]

不思考“它怎么做到的”就无法理解这个转换，是抽象思维能力不足，也是本科数学教育不足。实际上 `map` 的实现，完全可以是并行计算，或者分布式计算[^2]。

站在大多数普通程序员的角度，这个“不自然”的思维转换过程非常难受。另外，具备抽象思维能力，追求持续改进代码以符合“函数式”简洁美的程序员实在是凤毛麟角。还有，就是我们从小接受的计算机教育太根深蒂固，如果没有受过专门训练[^3]，就无法流畅自然的写出“函数式”代码。

写到这里，才发现我是旗帜鲜明的反对在团队开发中采用函数式编程的。因为程序员稀少，代码太抽象，思维非主流。但以发展的眼光来看，也许未来某天函数式编程会变得象今天的面向对象编程一样普及，成为人人必备的技能？

<br>
<br>
<br>
<br>
<br>

---
[^1]: 这里我想到的是饱受诟病的谭浩强C语言－[豆瓣：为什么那么多人讨厌谭浩强的书？](http://www.douban.com/group/topic/7198896/)
[^2]: 例如大数据处理模型 [MapReduce](http://en.wikipedia.org/wiki/MapReduce)
[^3]: 可以试试在线课程 [Functional Programming Principles in Scala](https://www.coursera.org/course/progfun)
