---
layout: post
title:  "并发设计 (3)"
date:   2015-06-21
summary: "异步与回调"
categories: blog
---

异步是实现可扩展系统，支持高并发，提升用户体验的必需手段。早期的异步API是用回调（Callback）来处理结果。如下例，当HTTP GET的结果返回时，我们提供的函数会被调用。值得注意：

* 提供的函数可以是一个闭包，捕获了上下文中的变量。
* 提供的函数是被“别人”调用的。可能是后台线程池里的一个线程。

{% highlight javascript %}
var name = "Jared";
getJSONFile("http://...", function(data) {
    var config = JSON.parse(data);
    // the closure can use `name`
});
{% endhighlight %}

考虑到错误处理，上面的代码会变得复杂，可读性变差。

{% highlight javascript %}
getJSONFile("http://...", function(err, data) {
    if (err) {
        // handle error, which is NOT the most important things.
    } else {
        // important things indent an extra level.
    }
});
{% endhighlight %}

考虑到更多现实情况，我们需要处理错误，超时，取消，成功四种。参数过多， 可以考虑把参数打包成一个Object。

{% highlight javascript %}
getJSONFile("http://...", function(err, isTimeout, isCanceled, data) {
});
{% endhighlight %}

当然，一个更好的重构是提供多个回调参数。

{% highlight javascript %}
getJSONFile("http://...", success, error, timedOut);
{% endhighlight %}

这个思想已经很接近Future模式了。

{% highlight javascript %}
var futureFile = getJSONFile(url);
futureFile.then(function(data) {
    // handle success
}).ifError(function(err) {
    // handle error    
}).setTimeout(2000).ifTimedout(function() {
    // handle timeout
});
// If canceled, none of above happens.
futureFile.cancel(); 
{% endhighlight %}

Future模式远比使用者看到的界面要复杂。

* 返回Future的函数都是异步函数。
* 这个函数的调用效果，相当于往消息队列里放一条消息。
* 消息的处理过程是在此地（inline）由所谓的combinator设定的。
* Combinator返回一个新的Future，于是可以把一组异步函数串联起来，形成continuation。
* 现代编程语言内建了async/await语法，用同步的写法来表达异步语义。

综合多种编程语言的文档和实现，才理解了Future这个目前最流行的异步编程理念。同时对Monad和Functional Thinking有了一点点体会。

> Always understand one level below your normal abstraction layer.
- Functional Thinking, Neal Ford

---

参考文献：

1.  JavaScript: [Promise of the Futures](https://www.youtube.com/watch?v=_ghej_y8Y5o)
2.  C++17: [I See a Monad in Your Future](https://www.youtube.com/watch?v=BFnhhPehpKw)
3.  Scala: [Futures and Promises](http://docs.scala-lang.org/overviews/core/futures.html)
4.  C#: [Asynchronous Programming with Async and Await](https://msdn.microsoft.com/en-us/library/hh191443.aspx)
5.  Python: [Coroutines with async and await syntax](https://www.python.org/dev/peps/pep-0492/)


