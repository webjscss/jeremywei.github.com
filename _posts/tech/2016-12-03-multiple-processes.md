---
layout: post
title: 操作系统多任务
tags: [tech]
---

![multiple-processes](http://{{ site.cdn }}/images/tech/multiple-processes.jpg "multiple-processes")

我们知道每个CPU在同一个时间内只能运行一个任务（进程），但是现实生活中我们需要计算机可以同时运行多个任务，
比如你在用文字编辑器写文章的时候想要同时听音乐。多进程实现了计算机同时执行多个任务的目标。
（注：本文以每台计算机只有一个CPU为例来进行讨论，多CPU或者多核以后再讨论）

## 多进程

多进程就是指操作系统把CPU的执行时间通过某种调度（scheduling）规则来进行分配，让多个进程都能获得执行的机会，
从而看起来好像多个进程在同时执行，实际上这是一种幻觉。

## 调度方式

### 合作式（Cooperative）

进程主动（定期或者空闲时候）让出CPU的执行时间给其他进程来执行，这种模式非常依赖于程序的设计，如果程序写的很差，
进程长时间占用CPU的执行时间（大量计算密集型的操作或者等待外设等），这会让其他进程没有机会执行，从而造成整个操作系统失去响应。

合作式多任务在早期的操作系统中使用得比较广泛（比如Windows 9x、[Classic Mac OS](https://en.wikipedia.org/wiki/Classic_Mac_OS)等等）。

### 抢占式（Preemptive）

操作系统把CPU的时间划分成一个一个的[时间片](http://stackoverflow.com/questions/16401294/how-to-know-linux-scheduler-time-slice)（毫秒级），抢占式多任务保证每个进程都能获得一个CPU时间片来执行。

什么叫做抢占呢？任何时间每个进程都可以被分为两种类型：I/O密集型（I/O bound）和CPU密集型（CPU bound）。CPU密集型的进程拥有比
I/O密集型进程拥有更高的优先级来得到CPU时间片，甚至从当前正在执行的I/O密集型进程手中抢来CPU时间。比如：

process a正在自己的时间片内执行，执行过程中需要等待外设，比如磁盘（I/O密集型），此时CPU的资源是被浪费的，而此时操作系统发现process b是个CPU密集型的进程，那么操作系统就可以把process a这个进程暂停，把这个CPU时间片交给process b来执行，这就是CPU抢占。

现代操作系统都支持抢占式多任务，包括Windows、macOS、Linux（包括Android）和iOS。

## 上下文切换

如果在当前时间片，CPU正在执行process a，当这个时间片结束之后，操作系统决定要让process b执行，那么操作系统将process a暂停，
并把process a的执行状态进行暂存，然后让process b在CPU上运行。当process b的时间片结束后，操作系统决定要让process a继续执行，
此时会把process a暂存的进程状态恢复，然后继续process a的执行。这种进程之间相互切换的过程叫做进程上下文切换。

由于时间片非常短，这种上下文切换的频率非常快，所以就给人一种错觉，好像多个进程在同时执行一样。

### 切换过程

![context-switch](http://{{ site.cdn }}/images/multiple-processes/context-switch.png "context-switch")

1. 当上下文切换发生的时候，操作系统把旧进程的状态进行保存，进程状态包括CPU寄存器、栈指针、[程序计数](https://en.wikipedia.org/wiki/Program_counter)以及其他操作系统数据。这些数据会被存储
在一个叫做[PCB](https://en.wikipedia.org/wiki/Process_control_block)（process control block）的数据结构中。

2. 把指向这个PCB的handle加入到ready queue队列中。

3. 当操作系统想要继续执行旧的进程，从ready queue中得到旧进程PCB的handle，然后从PCB中读取出进程状态，从而继续执行旧进程。

上下文切换会造成一定的系统开销，这种开销会在处理高并发的网络请求的时候产生性能问题。这也是Apache性能较Nginx差的[原因](/inside-nginx-how-we-designed-for-performance-scale.html)之一。


## 参考

* [https://en.wikipedia.org/wiki/Computer_multitasking](https://en.wikipedia.org/wiki/Computer_multitasking)
* [https://en.wikipedia.org/wiki/Preemption_(computing)](https://en.wikipedia.org/wiki/Preemption_(computing))
* [https://en.wikipedia.org/wiki/Context_switch](https://en.wikipedia.org/wiki/Context_switch)
