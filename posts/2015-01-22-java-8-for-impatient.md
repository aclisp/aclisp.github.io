# 《Java SE 8》读书笔记

这周看完[《写给大忙人看的Java SE 8》](http://it-ebooks.info/book/3677/)。

从完整度和深度来看，这本书的质量真的很高。全书最难理解的部分，如 Monadic Operations 和 Completable Futures，用了实例佐证抽象的概念，读起来很自然。可惜现在项目里使用的仍然是 Java 6……但愿我学到的不是屠龙之技。

下面是笔记：

# Lambda 表达式

前三章，都在传达这样一个意思：函数式编程已经来了。估计以后代码里会充斥着“不明觉厉”的 [closures](http://en.wikipedia.org/wiki/Closure_(computer_programming))，就像 Spring 官方文档 [Building REST services with Spring](http://spring.io/guides/tutorials/bookmarks/)。

Stream API 应该会在大数据量处理的情形下崭露头角，如大文件，数据库的处理。

Programming with Lambdas 谈到了 [Lazy Evaluation](http://en.wikipedia.org/wiki/Lazy_evaluation), [Currying](http://en.wikipedia.org/wiki/Currying), [Monad](http://en.wikipedia.org/wiki/Monad_(functional_programming))。这些属于基本上用不到的高阶技能。我觉得他们带来了新的代码组织方式和结构设计思路。另外，用 Java 这种 verbose 的语法来诠释这些概念，确实感觉更好理解。


# Concurrency 增强

值得一提的就是 Composing Asynchronous Operations。这简直是异步编程的春天。这段原文总结得非常好：

> This composability is the key aspect of the CompletableFuture class. Composing future actions solves a serious problem in programming asynchronous applications. The traditional approach for dealing with nonblocking calls is to use event handlers. The programmer registers a handler for the next action after comple- tion. Of course, if the next action is also asynchronous, then the next action after that is in a different event handler. Even though the programmer thinks in terms of “first do step 1, then step 2, then step 3,” the program logic becomes dispersed in different places. It gets worse when one has to add error handling. Suppose step 2 is “the user logs in”; then we may need to repeat that step since the user can mistype the credentials. Trying to implement such a control flow in a set of event handlers, or to understand it once it has been implemented, is challenging.
>
> With completable futures, you just specify what you want to have done, and in which order. It won’t all happen right away, of course, but what is important is that all the code is in one place.

看起来这套解决方案似乎更灵活一些。但我认为 C# 的 [async/await](https://msdn.microsoft.com/en-us/library/hh191443.aspx) 还是更加傻瓜一点。

# JavaScript Engine

> Nashorn is a pleasant environment for experimenting with the Java API.

这是实验 Java API 的最佳场所。而且可以用来代替 shell script。

其他部分主要是对文件、路径和目录的简便处理。

（完）
