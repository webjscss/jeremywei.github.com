---
layout: post
title: 深入NGINX - 我们如何面向性能和扩展性进行设计
tags: [tech]
---

![NGINX](http://{{ site.cdn }}/images/tech/nginx.jpg "NGINX")

原文：[https://www.nginx.com/blog/inside-nginx-how-we-designed-for-performance-scale/](https://www.nginx.com/blog/inside-nginx-how-we-designed-for-performance-scale/)

NGINX在性能上傲视群雄，这都归结于它的设计理念。很多web服务器和应用服务器都采用多线程或者基于进程的架构，NGINX通过采用复杂的事件驱动架构而脱颖而出，这种架构可以在当今的硬件上支持数十万的并发连接。

[《Inside NGINX》](http://www.nginx.com/resources/library/infographic-inside-nginx/?utm_source=inside-nginx-how-we-designed-for-performance-scale&utm_medium=blog)这个图表自上而下地从高层级的进程架构讲到了NGINX单个进程是如何处理多个连接。本文再更详细的介绍一下。

## 开场白 - NGINX的进程模型

![process-model](http://{{ site.cdn }}/images/nginx/process-model.png "process-model")

为了能更好的理解NGINX的设计，你需要知道NGINX是如何运行的。NGINX拥有一个主进程（master process，用来执行一些特权操作，比如读取配置和绑定端口号）、一定数量的工作进程（worker）和协助（helper）进程。

![process-list](http://{{ site.cdn }}/images/nginx/process-list.png "process-list")

在这个4核的服务器上，NGINX的主进程创建了4个工作进程和几个用来管理磁盘内容缓存的缓存协助进程（cache helper processes）。

## 架构为什么重要？

线程或者进程是任何Unix应用程序的基本组成单位。（从Linux操作系统的视角来说，线程和进程在很大程度上是一样的；最大的不同在于它们向其他详细共享内存的多少。）线程或者进程是一个完备的指令集合，操作系统可以把它调度到一个CPU核心上运行。大部分复杂的应用程序并行地执行多个线程或者进程有两个原因：

* 它们可以同时使用更多的CPU核心。
* 线程和进程能够使并行操作（举个例子，同时处理多个网络连接）变得很简单。

进程和线程是消耗资源的。它们每个都要使用内存和其他操作系统资源，并且它们需要从CPU核心上切换来切换去（这个操作叫作上下文切换）。大部分当今的服务器都能够同时处理上百个轻量级，活动的线程或进程，但是当内存耗尽或者高I/O负载引起大量上下文切换的时候，服务器的性能下降得很厉害。

网络应用常见的架构设计是为每个网络连接分配一个线程或者进程。这个架构很简单并且很容易实现，但是当应用需要处理成百上千并发连接的时候这个架构就无法扩展了。

## NGINX是如何工作的？

NGINX使用了一个理所当然的进程模型，来合理利用可用的硬件资源：

* 主进程（master process）执行一些特权操作，比如读取配置和绑定网络端口，然后创建少量的子进程（以下三种）
* 缓存加载进程（cache loader process）在启动的时候把磁盘上的缓存载入到内存中，然后退出。它的调用频率很低，所以资源需求也很少。
* 缓存管理进程（cache manager process）会定期执行，并且对磁盘缓存中的内容进行删减，以保证缓存的大小在配置范围内。
* 工作进程（worker processes）会做余下的所有事情！它们会处理网络连接，读写磁盘，并且和上游（upstream）服务器进行交互。

在大部分情况下，NGINX的推荐配置是 - 一个CPU核心上运行一个工作进程 - 最大化利用硬件的资源。你可以通过把配置项`worker_processes`的内容设置为`auto`来配置此功能：

```
worker_processes auto;
```

当NGINX服务器运行的时候，只有工作进程是繁忙的。每个工作进程以非阻塞（non-blocking）的方式来处理多个网络连接，从而减少了上下文切换的次数。

每个工作进程只有一个线程，并且彼此独立地处理网络连接。工作进程可以通过共享内存（shared memory）来实现缓存数据、持久化的会话数据以及其他其资源的共享。

## 剖析NGINX工作进程

![inside](http://{{ site.cdn }}/images/nginx/inside.png "inside")

每个NGINX工作进程以NGINX的配置信息进行初始化，然后主进程会把一些监听的套接字（socket）提供给它。

NGINX工作进程以等待监听套接字上的事件作为开始（[accept_mutex](http://nginx.org/en/docs/ngx_core_module.html?utm_source=inside-nginx-how-we-designed-for-performance-scale&utm_medium=blog&_ga=1.118629901.506501312.1479865310#accept_mutex)和[kernel socket sharding](http://www.nginx.com/blog/socket-sharding-nginx-release-1-9-1/?utm_source=inside-nginx-how-we-designed-for-performance-scale&utm_medium=blog)）。当有新的网络连接建立，事件就会被创建。这些连接会被分配到一个状态机（state machine）上 - HTTP状态机是用得最多的，但是NGINX也为流（原始TCP）处理以及邮件协议（SMTP, IMAP, and POP3）实现了状态机。

![state-machine](http://{{ site.cdn }}/images/nginx/state-machine.png "state-machine")

状态机本质上就是用来告诉NGINX如何处理请求的指令集合。大多数web服务器跟NGINX一样也使用类似的状态机 - 只是实现方式不同。

## 状态机调度

可以把状态机想像成是国际象棋的规则。每个HTTP的处理过程都是一局国际象棋比赛。棋盘一边是web服务器 - 一个可以快速作出决策的国际象棋大师。棋盘另一边是远程客户端 - 通过相对较慢的网络正在访问网站的web浏览器或者应用程序。

当然，这个游戏规则可能会非常复杂。比如，web服务器可能需要和其他服务（对上游应用进行代理）进行交互，或者需要到认证服务器进行认证。web服务器中的第三方模块也可以扩展这个游戏规则。

### 阻塞的状态机

![blocking-io](http://{{ site.cdn }}/images/nginx/blocking-io.png "blocking-io")

回想一下我们对进程或者线程的描述：可以被操作系统调度到CPU核心上进行执行的指令集合。大部分web服务器和web应用程序都使用`一个连接对应一个进程`或者`一个连接对应一个线程`的模式来玩儿这场游戏。每个进程或者线程都包含了从头到尾玩儿这个游戏所需的指令。在服务器执行进程的过程中，进程大部分的时间都是处于`阻塞（‘blocked’）`状态 - 等待客户端进行下一步操作。

1. web服务器进程在监听套接字上等待新的连接（客户端初始的新游戏）。
2. 当进程获取了新游戏，它开始玩儿这个游戏，为了等待客户端的响应而阻塞所有的操作。
3. 当游戏结束后，web服务器进程可能会等待一会看客户端是不是还想要再来一局（在持久（keepalive）连接的情况下）。如果连接关闭了（客户端没有响应或者超时），web服务器进程继续监听新的游戏。

需要记住的是每个活动的HTTP连接（每一局国际象棋比赛）都需要一个独立的进程或者线程（一个象棋大师）来处理。这个架构简单并且通过第三方的模块（新的比赛规则）很容易扩展。但是，这个方案有很大的问题：每个相当轻量级的HTTP连接用一个文件描述符和很少的内存就能表示，但是却要映射到一个独立的线程或者进程上，线程和进程非常耗操作系统资源。这样做编程很方便，但是却大量浪费资源。

### NGINX是真正的国际象棋大师

你可能听说过[同时进行的国际象棋表演赛](http://en.wikipedia.org/wiki/Simultaneous_exhibition?utm_source=inside-nginx-how-we-designed-for-performance-scale&utm_medium=blog)，即一个国际象棋大师同时跟n个对手下棋？

![Kiril Georgiev](http://{{ site.cdn }}/images/nginx/Kiril-Georgiev.gif "Kiril Georgiev")

Kiril Georgiev在索非亚（保加利亚）同时跟360个人一起下棋。最终284胜，70平，6败。

这就是NGINX工作进程下棋的方式。每个工作进程（记住 - 一般情况下每个CPU核心对应一个工作进程）都是一个可以通过进行几百场（实际上是几十万场）比赛的国际象棋大师。

![non-blocking-io](http://{{ site.cdn }}/images/nginx/non-blocking-io.png "non-blocking-io")

1. 工作进程等待来自被监听的套接字和已经建立连接的套接字发出的事件。
2. 套接字事件发生，然后工作进程处理它们：
    * 在被监听的套接字上发生的事件意味着客户端又开始了新一轮的国际象棋比赛。工作进程需要创建一个新的网络连接套接字。
    * 在已连接套接字上发生的事件意味着客户端走了一步棋。工作进程需要立刻进行响应。

工作进程从来不会因为网络而被阻塞，等待它的对手（客户端）进行响应。当它执行完之后，工作进程会立刻处理其他比赛中需要走的步，或者欢迎新玩家加入。

## 为什么这个架构比阻塞i/o，多进程的架构要快？

NGINX可以很好的扩展从而让每个工作进程支持几十万连接。每个新的连接会创建新的文件描述符并且消耗工作进程中很少的内存。每个连接的额外开销非常少。NGINX进程可以被固定在CPU上。上下文切换相对来说并不频繁，只有当进程没情可做的时候才会发生。

在阻塞i/o，每个进程处理一个连接的方案中，每个连接都需要大量的额外资源和开销，并且上下文切换（把CPU从一个进程切换到另一个进程）非常频繁。

想要更多的详细解释，可以看看这篇Andrew Alexeev写的关于NGINX架构的[文章](http://www.aosabook.org/en/nginx.html?utm_source=inside-nginx-how-we-designed-for-performance-scale&utm_medium=blog)，他是
NGINX Inc.的企业发展vp兼联合创始人。

经过适当的[系统调校](https://www.nginx.com/blog/tuning-nginx/?utm_source=inside-nginx-how-we-designed-for-performance-scale&utm_medium=blog)，NGINX每个工作进程可以处理几十万的并发HTTP连接，并且可以毫无压力的扛住流量峰值（新比赛像河水一样涌来）。

## 更新配置和升级NGINX

NGINX只拥有数量很少的工作进程，这种进程架构使得更新NGINX配置甚至其程序本身都非常的高效。

![reload-configuration](http://{{ site.cdn }}/images/nginx/reload-configuration.png "reload-configuration")

更新NGINX的配置是个非常简单，轻量级并且可靠的操作。这通常意味着运行`nginx –s reload`命令，这个命令会去检查磁盘上的配置，然后向主进程发送SIGHUP信号。

当主进程接收到SIGHUP信号，它会做两件事情：
1. 重新加载配置并创建一组新的工作进程。这些工作进程立刻开始接收网络连接并且处理请求（使用新的配置）。
2. 让旧的工作进程优雅的退出。工作进程停止接收新的连接。当每个HTTP请求处理完毕之后，工作进程关闭连接（这意味着不会存在长连接）。当所有的连接都关闭之后，工作进程退出。

重新加载配置的过程可能会使CPU和内存使用率出现一个小的峰值，但是通常情况下跟活动连接的资源负载相比微乎其微。你可以每秒重新加载配置多次（事实上很多NGINX使用者就是那么做的）。当很多批NGINX工作进程在等待连接被关闭的时候会发生一些问题，但是很少见，即使出现也很快就能解决。

NGINX二进制程序的升级过程把高可用发挥到了极致 - 你可以迅速的完成升级，并且不会导致任何连接丢失、宕机或者服务中断。

![upgrade-binary](http://{{ site.cdn }}/images/nginx/upgrade-binary.png "upgrade-binary")

二进制程序的升级过程和优雅的重新加载配置的方案是相似的。NGINX新的主进程会和原始的主进程同时运行，并且他们之间共享监听的套接字。这两个进程都处于活动状态，并且它们各自的工作进程都可以处理网络请求。之后就可以向旧的主进程和其工作进程发送信号，使其优雅的退出。

[《Controlling NGINX》](http://nginx.org/en/docs/control.html?utm_source=inside-nginx-how-we-designed-for-performance-scale&utm_medium=blog&_ga=1.98190747.506501312.1479865310)这篇文章中更加详细的描述了这整个过程。

## 总结

[《Inside NGINX infographic》](http://www.nginx.com/resources/library/infographic-inside-nginx/?utm_source=inside-nginx-how-we-designed-for-performance-scale&utm_medium=blog)这篇文章从高视角简述了NGINX的工作原理，但是简单解释的背后是十年以上的创新和优化工作，使NGINX可以在大范围的硬件上表现出最好的性能，并且要同时保持现代web应用需要的安全和可用性。

如果你想要了解更过关于NGINX的优化方法，看看这些不错的资源：

* [Installing and Tuning NGINX for Performance](http://www.nginx.com/resources/webinars/installing-tuning-nginx/?utm_source=inside-nginx-how-we-designed-for-performance-scale&utm_medium=blog) (webinar; slides at Speaker Deck)
* [Tuning NGINX for Performance](http://www.nginx.com/blog/tuning-nginx/?utm_source=inside-nginx-how-we-designed-for-performance-scale&utm_medium=blog)
* [The Architecture of Open Source Applications – NGINX](http://www.aosabook.org/en/nginx.html?utm_source=inside-nginx-how-we-designed-for-performance-scale&utm_medium=blog)
* [Socket Sharding in NGINX Release 1.9.1](http://www.nginx.com/blog/socket-sharding-nginx-release-1-9-1/?utm_source=inside-nginx-how-we-designed-for-performance-scale&utm_medium=blog) (使用SO_REUSEPORT套接字配置项)
