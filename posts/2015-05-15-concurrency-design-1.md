# 并发设计 (1)

受到“[能利用爬虫技术做到哪些很酷很有趣很有用的事情？](http://www.zhihu.com/question/27621722)”一文的影响，也打算自己实现一个爬虫。一个高效的爬虫程序必然是一个并发程序。在计划如何实现这个程序的过程中，我有了以下思考。

## 编程语言

为了效率和节省资源，选了Go。

* 首先，需要一种静态强类型语言，编译期就能发现大多数错误。Python之流马上出局。
* 其次，不能用基于JVM的语言，因为打算放在日常PC后台运行，内存用的越少越好。
  - 实际上Scala是一种设计极好的语言，但这次不用它。
* 最终，可选的有
  - C++，老牌语言。
  - Go，工程语言。带GC。
  - Rust，新生语言。问题是太新，1.0版本对并发几乎没有语言级别的支持。设计上媲美Scala。

## 编程框架

爬虫本质上是一个网络客户端程序。节省资源起见，肯定是用异步IO。平台中立的C++编程框架也有不少，cppreference上有一个[列表](http://en.cppreference.com/w/cpp/links/libs)。

* Boost 概念新颖，但是太费脑了，项目也不容易维护。
* ACE 基本没有封装，写起来细节太多太烦。
* POCO 只有一个可怜的Reactor，不支持真异步。

C++的话，必然选Qt。其文档完备，类库丰富，接口人性化。最重要的，Qt程序必然自带事件循环，自然的，所有组件的调用方式必然支持真异步。

## HTML解析

爬虫一个最重要的功能就是抓取超链接。

Qt解析HTML有点麻烦，要用第三方的TidyLib[^QtHTML]。Go有HTML5解析器的[实现](http://godoc.org/golang.org/x/net/html)。

## 异步IO详解

何谓“真异步”？[^TrueAsync] 谈这个得先说IO的三种方式：

* __同步阻塞__：发起系统调用之后，调用者被阻塞，直到系统调用完成。
  - 最常用，但效率最低。
  - 通常使用场景为RPC。
  - 要实现并发，则必须上多线程或线程池。
  - 根据需要可以引入由系统控制的超时机制：
    + 超时无穷大，则系统调用必须拿到结果才返回。例如，读取磁盘文件。
    + 超时设为某有限值，则系统调用有可能失败。例如，建立TCP连接。
    + 超时为0，退化为下面的“同步非阻塞”。
* __同步非阻塞__：就是上面的“同步阻塞，超时为0”。
  - 通常用单线程来做IO多路复用（Multiplexing）。
  - 必须采用特殊的系统调用（比如select）来做调度。
  - 天生支持并发，但并发数受限于调度机制。（如select有1024个描述符的限制。）
  - 上层应用协议的超时机制，只能程序员自己实现。
  - 古老的Unix服务器基本都用这种方式实现。
* __异步非阻塞__：发起系统调用之后，立即返回。
  - 请求由OS提供资源和线程来处理；应答同样是由OS来处理。
  - OS处理完成时，调用者收到（异步）通知，以及由OS准备好的数据。调用者可以接着发起下一次系统调用。
  - 必须采用特殊的系统调用（比如io_getevents）来做调度
  - 天生支持并发。由于读写均在OS内核完成，并发数只受限于系统资源（CPU，内存）。
  - 上层应用协议的超时机制，只能程序员自己实现。
  - 显而易见，这是性能和扩展性最好的方式。也是所谓的“真异步”。

## 反应堆模式

非阻塞的编程模型非常复杂，对程序员非常的不友好。于是后人把它们封装并抽象成两种模式：

* Reactor
* Proactor

两者的区别是：

* 前者在event handler里做实际的读写操作；后者由OS做实际的读写操作，读写完成后event handler才进入角色。
* 前者在用户空间做读写操作；后者在内核空间做读写操作。
* 前者由用户来控制并发；后者由内核自动控制并发。

值得注意的是，Proactor可以用Reactor来实现。这样的话，等同于Proactor框架模拟内核来做了IO的读写操作。于是，用户界面是统一的。流行的异步IO库必须做了这样的封装。[^BoostAsio]

如果选C++，那就只能到此为止。可以想象，用VS+Qt做开发的样子。但问题是Qt库太大了……而且编程语言发展到今天，我们已经有了更好的方式来支持并发。

## 并发

即使反应堆模式给非阻塞的编程模型带来了极大的简化，人们总是不满足的。反应堆模式的问题是会造成callback hell。Qt的signal/slot机制已经做得很好，但在今天这个新语言井喷得年代，把并发做到语言内建（或库）支持才是王道。

并发的处理总结起来有这么几种：

* 协程（[Coroutine](http://en.wikipedia.org/wiki/Coroutine)） 最著名的是C#和Python的async/await语法，同步的写法来实现异步。
* [LWP](http://en.wikipedia.org/wiki/Light-weight_process) 最著名的是goroutine。
* [Actor](http://en.wikipedia.org/wiki/Actor_model) 由Erlang和Scala采用。

不得不说，Go的思路真是非常的工程化[^AsyncGo]和简单粗暴：咱就用同步的方式来写，如果系统支持不了这么多线程，那就让语言运行时搞定；作为用户没有必要知道太多，异步……那是什么？

对于Actor模型，我的理解它是自治的，封闭的。没有办法与已有的IO原语集成。换句话说，IO仍然要借助其他模型。只是在内部处理上，可以实现Parallel Pipelines模式。

---
[^TrueAsync]: [Reactor and Proactor: two I/O multiplexing approaches](http://www.artima.com/articles/io_design_patterns2.html)
[^BoostAsio]: [Proactor and Boost.Asio](http://www.boost.org/doc/libs/1_58_0/doc/html/boost_asio/overview/core/async.html)
[^AsyncGo]: [Async IO Part 1 – Go vs. Node.js](http://www.reddit.com/r/golang/comments/25iic3/async_io_part_1_go_vs_nodejs/)
[^QtHTML]: [Handling HTML in Qt](https://wiki.qt.io/Handling_HTML)
