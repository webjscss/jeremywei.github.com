---
layout: post
title: 设计一个务实的RESTful API
tags: [tech]
---

![REST](http://{{ site.cdn }}/images/tech/rest.jpg "REST")

##TL;DR

* [API是开发者的用户界面 - 所以多花点功夫使其更友好](#requirements)
* [使用RESTful的URL和action](#restful)
* [所有地方都要使用SSL，没有例外](#ssl)
* [一个API的好用与否取决于它的文档 - 所以文档要好好写](#)
* [通过URL来控制版本，不要通过header](#)
* [使用查询参数来实现高级功能：过滤、排序和搜索](#)
* [提供方法用来限制从API返回的字段](#)
* [对于POST、PATCH和PUT的请求返回一些有用的东西](#)
* [HATEOAS当前还不太实用](#)
* [尽量使用JSON，迫不得已的时候使用XML](#)
* [理论上你应该在JSON中使用驼峰格式，但是下划线格式的可读性比驼峰格式要高20%](#)
* [默认情况API要输出完整的内容，并且要确认支持gzip](#)
* [默认情况下不要在响应内容的外边再套一层大括号](#)
* [考虑对POST、PUT和PATCH等请求的body使用JSON格式](#)
* [使用Link header分页](#)
* [提供一种方法来自动加载相关资源的内容](#)
* [提供一种方法来覆盖HTTP方法](#)
* [提供响应header用来对请求频次进行限制](#)
* [使用token来进行普通验证，当需要用户授权的时候使用OAuth2](#)
* [要在响应header中包含缓存所使用的header](#)
* [定义一个容易理解的错误格式](#)
* [高效的使用HTTP状态码](#)

<span id="requirements"></span>
##API的关键要求
网上很多关于API设计的意见都是一些学术讨论，里边充斥着对模糊标准非常主观的解释，而不是讨论在现实世界中如何落地。这篇文章的目标是描述一个最佳实践：如何为当今的web应用设计一个务实的API。如果一个标准不合理，我不会去尝试满足这个标准。为了帮助决策过程进行，我写下了一些API必须要满足的要求：

* 它应该使用合理的web标准
* 它应该对开发者友好，并且可以通过浏览器的地址栏就能浏览其功能
* 它应该是简单、直观并且一致的
* 它应该是高效，并且要跟其他要求保持平衡

API是开发者使用的UI - 就像任何UI一样，保证用户体验是很重要的。

<span id="restful"></span>
##使用RESTful的URL和action

如果说有一个理念被广泛的采用，那就是RESTful原则。这些原则由Roy Fielding在其博士论文
[《network based software architectures》](http://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm)的[第五章](http://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)中被首次提出。

REST的关键原则涉及到要把你的API分解成一种逻辑上的资源。这些资源可以通过有具体含义的HTTP方法（GET, POST, PUT, PATCH, DELETE）来进行修改。

但是我该怎么定义一个资源呢？当然，这些资源应该是名词（不是动词！），这样对API消费者来说更容易理解。尽管你内部的模型跟资源之间是一对一的映射关系，但这不是必须的。这里所说的重点是不要把内部实现的细节暴露到你的API之中。

一旦定义好了你的资源，你需要指明对这些资源都可以做什么操作以及如何通过你的API来实现。RESTful原则提供了一个使用HTTP方法来实现CRUD操作的策略，如下：

* GET /tickets - Retrieves a list of tickets
* GET /tickets/12 - Retrieves a specific ticket
* POST /tickets - Creates a new ticket
* PUT /tickets/12 - Updates ticket #12
* PATCH /tickets/12 - Partially updates ticket #12
* DELETE /tickets/12 - Deletes ticket #12

REST牛逼的地方在于你只需要在/tickets这个单一入口上利用已经存在的HTTP方法来实现有意义的功能。除此之外你不需要额外的方法命名规则，并且URL结构也很清晰。REST万岁！

接入端的名字应该是单数还是复数呢？这里我们遵循keep-it-simple原则。尽管有人会告诉你使用复数来描述一个资源实例是错的，但是务实的做法是为了保持URL格式的一致，都要使用复数。不需要处理odd pluralization问题 (person/people, goose/geese) 对于API消费者来说更友好，并且对于API提供者来说也更容易实现（因为大部分现代框架默认情况下会在同一个普通的controller中同时处理/tickets和/tickets/12这类的请求）。

但是如何处理资源之间的关系呢？如果关系只存在于另一个资源，RESTful原则提供了一个有用的指引。让我们通过一个例子来看一下。Enchant中的一个ticket由很多message构成。这些message可以通过以下形式逻辑性地映射到/tickets这个接入端口上：

* GET /tickets/12/messages - Retrieves list of messages for ticket #12
* GET /tickets/12/messages/5 - Retrieves message #5 for ticket #12
* POST /tickets/12/messages - Creates a new message in ticket #12
* PUT /tickets/12/messages/5 - Updates message #5 for ticket #12
* PATCH /tickets/12/messages/5 - Partially updates message #5 for ticket #12
* DELETE /tickets/12/messages/5 - Deletes message #5 for ticket #12

或者，如果关系可以独立于资源存在，那么在资源内容的输出中只包含关系id就可以了。之后API消费者不得不再去访问关系的入口。不过，如果这个关系跟资源存在一起，那么API可以提供一个功能，自动把这个关系的内容嵌入其中，从而避免重复访问API。

对于不能映射成CRUD操作的action该怎么办？
这是容易引起困惑的地方。这里有几个解决方案：

1. 把action映射成一个资源的一个字段。如果这个action没有参数的这么做是可以的。举个例子，`activate`这个action可以映射成一个boolean类型的`activated`字段，然后可以通过PATCH方法来更新这个资源。

2. 在RESTful原则下，把它当作的子资源（sub-resource）来对待。比如，GitHub的API允许你通过`PUT /gists/:id/star`来star一个gist，通过`DELETE /gists/:id/star`来unstar。

3. 有时候你真的没有办法把action映射到一个合理的RESTful结构上。比如，通用搜索这个操作就没法被映射到一个指定的资源入口上。这种情况下， /search是最好的选择，即使它不是一个资源。这没什么问题 - 尽可能做对API消费者有益的设计，并确保有清晰的文档说明，以免造成困惑。

<span id="ssl"></span>

<span id="introduction"></span>

##参考
* [REST 架构该怎么生动地理解？](https://www.zhihu.com/question/27785028)
