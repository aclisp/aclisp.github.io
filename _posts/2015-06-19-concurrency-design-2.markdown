---
layout: post
title:  "并发设计 (2)"
date:   2015-06-19
categories: blog
---

这里总结一下常见的几种异步模式。

# HTTP的异步请求处理

HTTP通信经过了这些演化：

*   1.0 到 1.1。
    最重要的变化是连接保持 （persistent connections）。1.0时代，连接是短暂的，在一次请求应答之后即关闭。1.1的连接保持的特性降低了通信延迟。

*   每个连接由单独线程处理。
    这是一些Web服务器的通常做法。缺点是一旦用户增长，连接数变得很大，服务器资源消耗严重。同时，由于单个用户的请求通常是零星地，连接处理线程大部分时间在发呆，资源闲置也很严重。

*   每个请求由单独线程处理。
    这是流行的Web服务器的做法。所有的连接统一由反应堆轮询管理。请求事件会被送到相应的处理函数，并从业务线程池选取一个线程来执行。完美解决资源消耗和闲置的问题。

*   Ajax时代。
    随着越来越多的网站用Ajax技术来提升用户体验，Web服务器的并发请求数也越来越大。虽然Web服务器的接入层实现本身没问题，但是业务逻辑层会有严重缺陷。请求事件的处理函数里，访问数据库是同步的，访问WebService也是同步的。这些访问一旦阻塞，就导致业务线程池里大量线程闲置。

*   服务端推送。
    要在HTTP的请求应答模型（不用WebSocket）下实现服务端推送，客户端（浏览器里的Ajax JavaScript）通常会这么搞：发送请求，等待服务器应答；一旦收到应答（或超时），马上再发送请求，继续等待应答；...服务器总是有一个待处理的请求，但是它不能立刻（在请求处理函数返回前）应答；只有当推送事件发生时，才允许应答。

为了实现服务端推送的异步事件，也为了让同步访问数据库和WebService不占用处理HTTP请求的线程池，Java Servlet 3.0 引入了异步模式。这个模式的本质是：__在请求处理函数中提交一段回调逻辑，延迟写和关闭HttpServletResponse__。

当然，任何语言实现的Web框架都可以借鉴这种模式。有一点值得注意，访问数据库和WebService始终是同步的，不占用处理HTTP请求的线程池，就要占用其它某个线程池，除非 __用异步来访问数据库和WebService__。

下篇会讲一讲这种异步接口的设计和实现。

---

参考文献：

1.  [Asynchronous Processing Support in Servlet 3.0](http://www.javaworld.com/article/2077995/java-concurrency/asynchronous-processing-support-in-servlet-3-0.html)
2.  [Asynchronous Servlets in Servlet Spec 3.0](http://www.softwareengineeringsolutions.com/blogs/2010/08/13/asynchronous-servlets-in-servlet-spec-3-0)
3.  [Asynchronous Request Processing in Spring Framework](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#mvc-ann-async)
