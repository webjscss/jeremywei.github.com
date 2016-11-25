---
layout: post
title: 设计一个务实的RESTful API
tags: [tech]
---

![REST](http://{{ site.cdn }}/images/tech/rest.jpg "REST")

原文：[http://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api](http://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api#json-requests)

##   TL;DR

* [API是开发者的用户界面 - 所以多花点功夫使其更友好](#requirements)
* [使用RESTful的URL和action](#restful)
* [所有地方都要使用SSL，没有例外](#ssl)
* [一个API的好用与否取决于它的文档 - 所以文档要好好写](#docs)
* [通过URL来控制版本，不要通过header](#versioning)
* [使用查询参数来实现高级功能：过滤、排序和搜索](#advanced-queries)
* [提供方法用来限制从API返回的字段](#limiting-fields)
* [对于POST、PATCH和PUT的请求返回一些有用的东西](#useful-post-responses)
* [HATEOAS当前还不太实用](#hateoas)
* [尽量使用JSON，迫不得已的时候使用XML](#json-responses)
* [理论上你应该在JSON中使用驼峰格式，但是下划线格式的可读性比驼峰格式要高20%](#snake-vs-camel)
* [默认情况API要输出完整的内容，并且要确认支持gzip](#pretty-print-gzip)
* [默认情况下不要在响应内容的外边再套一层大括号](#envelope)
* [考虑对POST、PUT和PATCH等请求的body使用JSON格式](#json-requests)
* [使用Link header分页](#pagination)
* [提供一种方法来自动加载相关资源的内容](#autoloading)
* [提供一种方法来覆盖HTTP方法](#method-override)
* [提供响应header用来对请求频次进行限制](#rate-limiting)
* [使用token来进行普通验证，当需要用户授权的时候使用OAuth2](#authentication)
* [要在响应header中包含缓存所使用的header](#caching)
* [定义一个容易理解的错误格式](#errors)
* [高效的使用HTTP状态码](#http-status)

<span id="requirements"></span>
##    API的关键要求
网上很多关于API设计的意见都是一些学术讨论，里边充斥着对模糊标准非常主观的解释，而不是讨论在现实世界中如何落地。这篇文章的目标是描述一个最佳实践：如何为当今的web应用设计一个务实的API。如果一个标准不合理，我不会去尝试满足这个标准。为了帮助决策过程进行，我写下了一些API必须要满足的要求：

* 它应该使用合理的web标准
* 它应该对开发者友好，并且可以通过浏览器的地址栏就能浏览其功能
* 它应该是简单、直观并且一致的
* 它应该是高效，并且要跟其他要求保持平衡

API是开发者使用的UI - 就像任何UI一样，保证用户体验是很重要的。

<span id="restful"></span>
##   使用RESTful的URL和action

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
##   所有地方都要使用SSL - 没有例外

要一直使用SSL。没有例外。如今，你的web API可以从任何有互联网的地方（像图书馆，咖啡馆，机场等等）被访问到。这些地方并不都是安全的。很多地方根本没有对网络连接进行加密，如果验证凭证被劫持的话，这样很容易被窃听或者被冒充。

一直使用SSL的另一个优势是，加密的连接简化了用户验证的工作 - 你可以使用简单的access token，而不需要对每个API请求进行签名。

需要注意的一件事是以非SSL的形式访问API的URL。不要把请求跳转到它们的SSL版本上。直接抛出一个严重错误！

<span id="docs"></span>
##   文档

一个API的好用与否取决于它的文档。文档应该很容易被找到并且能够公开访问。大部分开发者在尝试进行任何集成之前都会先看看文档。当文档被隐藏在一个PDF文件里或者需要登录才能查看的时候，文档不仅很难被找到而且也很不容易被检索。

文档应该有一些完整的展示[请求/响应]过程的例子。请求的例子应该是可以直接粘贴使用的 - 链接可以直接被粘贴到浏览器或者curl例子可以被粘贴到终端里执行。GitHub和Stripe在这方面做得就很好。

一旦你发布一个公共API，实际上你已经承诺不会在没有通知的情况下使其不可用。文档中API更新的地方必须包含废弃功能的时间表以及细节。更新应该通过博客（比如changelog）或者通过邮件列表来发布（二者都有的话就更完美了）。

<span id="versioning"></span>
##   版本控制

你要始终对自己的API进行版本控制。版本控制可以帮助你更快的迭代，并且可以防止无效的请求访问新的API。这还可以帮助你平滑过渡任何大版本的升级，在这个过程中你还可以继续提供老版本的API一段时间。

[API的版本信息究竟是应该包含在URL里边还是header里边](http://stackoverflow.com/questions/389169/best-practices-for-api-versioning)，围绕着这个目前存在分歧。从学术上来讲，版本信息应该放在header里边。但是为了保证能够通过浏览器去浏览资源的不同版本，版本信息应该放在URL里边（还记得在文章开始的地方指定的对API的要求吗？）。

我是Stripe所采用的API版本控制方式的拥趸 - URL包含大版本号（v1），但是API还有基于日期的子版本，可以使用一个自定义的HTTP header来选择。在这种情况下，大版本作为一个整体保证了API数据结构的稳定，同时子版本可以用来交付一些小的更新（比如废弃字段，接入端变更等等）。

一个API永远不会完全稳定。变化是必然的。重点是变化该如何管理。对于大部分API来说，优秀的文档和提前声明的定时更新计划是可以接受的方案。对于行业和API的潜在消费者来说这么做是比较靠谱的。

<span id="advanced-queries"></span>
##  结果过滤、排序以及搜索

最好尽可能保持资源的基础URL精简。复杂的结果过滤、排序等要求和高级搜索（当被限制在一个单独的资源上）都可以通过简单得在基础URL上添加查询参数来实现。让我们看看具体例子：

过滤：为每个实现了过滤功能的字段都提供一个唯一的查询参数。举个例子，当从`/tickets`请求一个ticket列表的时候，你可能只想要state是open的那些。这可以通过类似于`GET /tickets?state=open`这类请求来实现。我们在这里通过state查询参数实现了一个过滤器。

排序：跟过滤很像，我们可以用一个通用的`sort`参数来描述排序规则。为了满足复杂的排序要求，sort参数可以接收以逗号分隔的字段列表，每个字段可以负号来表示降序。让我们来看一些例子：

* GET /tickets?sort=-priority - 获取ticket的列表，按照priority降序排列
* GET /tickets?sort=-priority,created_at - 获取ticket的列表，先按照priority降序排列，然后再按照created_at升序排列。

搜索：有时候基本的过滤器功能不够，你需要全文搜索功能。可能你正在使用ElasticSearch或者其他基于Lucene的技术。当全文搜索成为获取某类资源信息的方法的时候，这个功能就可以通过资源API的查询参数提供出来。比如`q`。搜索的查询条件应该直接传给搜索引擎，API返回的格式应该跟普通列表查询一样。

把以上这些合并在一起，我们可以得到以下查询：

* GET /tickets?sort=-updated_at - 获取最近更新的ticket
* GET /tickets?state=closed&sort=-updated_at - 获取最近关闭的ticket
* GET /tickets?q=return&state=open&sort=-priority,created_at - 获取提到了"return"这个词的优先级最高的ticket

对常用的查询设置别名

为了给用户提供更好的API体验，可以考虑把多个查询条件合并成更容易访问的RESTful路径。比如，上面提到过的查询最近关闭的ticket列表就可以被合并成`GET /tickets/recently_closed`

<span id="limiting-fields"></span>
##  对API返回的字段进行限制

API消费者并不都一直需要资源的所有内容。提供选择返回字段的能力对API消费者最小化网络流量，并且加快他们对API的调用大有益处。

可以使用`fields`查询参数，这个参数的内容是以逗号分隔的字段列表。举个例子，下面的请求将会获取到足够多的信息来展示状态为open的ticket的排序列表：

* GET /tickets?fields=id,subject,customer_name,updated_at&state=open&sort=-updated_at

<span id="useful-post-responses"></span>
##  更新和创建操作应该返回资源内容

PUT、POST或者PATCH操作可能会对资源的底层字段进行修改，这些字段并没有在API中提供（比如created_at或者updated_at这类的时间戳）。为了防止API消费者重复更新资源，这个API要在响应中包含updated（或者created）字段。

如果是用POST来创建一个资源，那么返回[201状态码](http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html#sec9.5)并在[Location header](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.30)中指明新资源的URL。

<span id="hateoas"></span>
##   你该用HATEOAS吗？

对于到底是API消费者自己生成链接还是API提供链接，有很多不同的意见。RESTful设计原则提出了[HATEOAS](https://blog.apigee.com/detail/hateoas_101_introduction_to_a_rest_api_style_video_slides)，其大体上的意思是：与API的交互动作应该由API返回的metadata提供，而不是基于外部信息。

尽管web已经工作在HATEOAS原则上（比如我们访问一个网站的首页以及根据我们当前访问的内容自动生成的其他链接），但是我并不认为我们现在在API上使用HATEOAS是合适的。当浏览一个网站的时候，该点击什么链接是在运行的时候决定的。但是，对于API来说，该发起什么请求是在进行API集成的时候决定的，而不是运行的时候。这个决定能延迟到运行的时候再做吗？当然可以，但是通过代码来控制该调用什么API仍然没法解决在不停机的情况下应对API更新的问题，所以这么做没有多大益处。这么说来，我认为HATEOAS只是一个承诺，但现在的时机并不成熟。还有许多标准需要指定，许多工具需要开发才能让人认识到它的潜力。

现在看来，最好在API的响应内容中包含资源标识符，以便API消费者用来创建以后需要访问的链接。使用标识符有很多优势 - 这可以是网络传输的数据最小化，并且API消费者需要存储的数据也被最小化了（他们只存储标识符而不是存储包含标识符的整个URL）。

还有一点，这篇文章提倡把版本号放在URL里边，API消费者存储资源标识符而不是URL从长远来看是合理的。毕竟，无论版本如何变化标识符是稳定的，而URL则不是。

<span id="json-responses"></span>
##  响应格式只使用JSON

是时候在API中放弃XML了。XML臃肿，难解析，难读，并且它的数据模型和大部分编程语言的数据模型不兼容，而且它的扩展优势基本用不上，因为你主要的需求是把资源的内部表现形式以一种序列化的形式输出。

我不会花功夫再解释以上观点了，因为好像很多公司([YouTube](http://apiblog.youtube.com/2012/12/the-simpler-yet-more-powerful-new.html), [Twitter](https://dev.twitter.com/docs/api/1.1/overview#JSON_support_only) & [Box](http://developers.blog.box.com/2012/12/14/v2_api/))都开始抛弃XML了。

我只给你们看看Google Trends的图表([XML API vs JSON API](http://www.google.com/trends/explore?q=xml+api#q=xml%20api%2C%20json%20api&cmpt=q))用来思考：

![ddd](http://www.vinaysahni.com/images/201305-xml-vs-json-api.png)

不过，如果你的客户中存在大量的企业客户，那么你将不得不支持XML。如果你必须这么做，你将会面临一个新的问题：

数据格式的选择到底是通过Accept header实现还是通过URL实现？为了能够通过浏览器查看，应该通过URL来实现。最合理的方法是在URL后面加一个.json或.xml扩展名。

<span id="snake-vs-camel"></span>
##  字段名称采用下划线还是驼峰命名方式

如果你在使用JSON(JavaScript Object Notation)作为你API的主要输出格式，那么「正确」的做法是遵循JavaScript的命名规则 - 这意味着字段名要使用驼峰方式命名！如果你又去为各种语言开发客户端库的话，你最好使用每种语言各自的命名惯例 - C#和Java使用驼峰命名方式，python和ruby使用下划线命名方式。

深思：我一直觉得下划线命名方式比JavaScript的驼峰命名方式要更易读。只是我没有任何证据来支持我的内心感觉，直到现在为止。基于2010年[《eye tracking study on camelCase and snake_case》 (PDF)](http://ieeexplore.ieee.org/xpl/articleDetails.jsp?tp=&arnumber=5521745)研究，下划线格式比驼峰格式的可读性要高20%！这个对可读性的重要影响将会影响到API的可读性以及文档中的例子。

很多流行的JSON API都是使用下划线命名方式。我怀疑这么做的原因可能是因为序列化库使用了底层语言的命名规则。可能我们需要JSON序列化库可以处理命名规则转换。

<span id="pretty-print-gzip"></span>
##  默认情况下不要过滤API输出中的空格，并且要支持gzip

如果对API的输出结果进行空格压缩，从浏览器访问的话，体验很差。即使通过一些查询参数（比如?pretty=true）可以输出不过滤空格的内容，但是对于一个API来说默认输出未过滤空格的内容是更友好的。额外增加的数据传输是微不足道的，特别是当开启gzip之后。

考虑一些使用场景：如果一个API消费者正在debug，并在代码中把从API接收到的数据打印了出来 - 默认情况下必须是可读的。这些是小事，但是这些小事可以使API用起来更舒服。

额外的数据传输到底有多大？

让我们看一个真实的例子。GitHub的API默认情况下是不压缩空格的，我从中拉下来了一些数据。我也会做一些gzip方面的对比：

```
$ curl https://api.github.com/users/veesahni > with-whitespace.txt
$ ruby -r json -e 'puts JSON JSON.parse(STDIN.read)' < with-whitespace.txt > without-whitespace.txt
$ gzip -c with-whitespace.txt > with-whitespace.txt.gz
$ gzip -c without-whitespace.txt > without-whitespace.txt.gz
```
这几个文件的大小如下：

* without-whitespace.txt - 1252 bytes
* with-whitespace.txt - 1369 bytes
* without-whitespace.txt.gz - 496 bytes
* with-whitespace.txt.gz - 509 bytes

在这个例子中，当没开启gzip的时候，空格在输出内容中的大小占8.5%，当gzip开启的时候占2.6%。换句话说，gzip压缩可以节省60%的带宽。既然保留输出结果中的空格的成本相对来说很小，那么最好默认情况下保留空格并且开启gzip压缩！

为了巩固这个观点，Twitter发现当在其[Streaming API](https://dev.twitter.com/docs/streaming-apis)上开启gzip压缩的时候能[节省80%的流量](https://dev.twitter.com/blog/announcing-gzip-compression-streaming-apis
)（某些情况下）。Stack Exchange做得更绝，[从不返回一个未经压缩的响应内容](https://api.stackexchange.com/docs/compression)。

<span id="envelope"></span>
##  默认情况下不要再外层套一个大括号，但是当需要的时候可以加

很多API都会像下面一样在它们的响应内容外边套一个大括号：

```
{
  "data" : {
    "id" : 123,
    "name" : "John"
  }
}
```
这么做有很多理由 - 这使得在响应中添加额外的元数据（metadata）或分页信息很容易，因为一些REST客户端访问HTTP header不是很容易，并且JSONP请求根本无法访问HTTP header。随着像[CORS](http://www.w3.org/TR/cors/)和[RFC 5988中的Link header](http://tools.ietf.org/html/rfc5988#page-6)等标准的快速通过，这么做变得没有任何必要。

我们默认情况下还是不要在外边嵌套大括号，除了一些特殊情况。

特殊情况下该如何嵌套大括号呢？

两种情况下需要嵌套大括号 - 如果API需要支持JSONP跨域访问或者客户端无法访问HTTP header。

JSONP的请求跟着一个额外的查询参数（通常叫做callback或者jsonp），用来表示回调函数的名字。如果这个参数传了的话，API应该在响应内容的外层嵌套大括号，通常都是返回200状态码，然后把真正的状态码放在JSON内容中。任何其他需要随着响应内容返回的HTTP header应该放在JSON字段里边，像这样：

```
callback_function({
  status_code: 200,
  next_page: "https://..",
  response: {
    ... actual JSON response body ...
  }
})
```
类似地，如果想要支持受限的HTTP客户端，可以提供一个额外的查询参数?envelope=true，这将会在响应内容外层嵌套大括号（没有JSONP回调函数）。

<span id="json-requests"></span>
##  对POST、PUT和PATCH方法的body使用JSON格式

如果你正在使用本文所介绍的方案，那么你的API输出的内容都是JSON格式。让我们考虑考虑在API输入的地方也使用JSON。

大部分API在请求body中都使用URL编码规则。URL编码规则跟它的字面意思一样 - 请求body中的键值对所使用的编码规则跟URL查询参数使用的一样。这个编码规则简单，支持的范围广并且能把事情搞定。

但是，URL编码存在一些问题。它没有数据类型的概念。这会强制API把整型和布尔类型转换成字符类型。而且，它没有真正的层级结构体的概念。尽管有一些惯例可以从键值对构建出一些结构体（比如在key后面追加一个[]符号来表示一个数组），但是这不能跟JSON原生的层级结构体相提并论。

如果API很简单，URL编码足够。但是，复杂的API应该使用JSON作为API的输入。不管怎么样，选择一个编码然后保证整个API都用这个就好了。

一个API如果接受JSON编码的POST、PUT和PATCH请求，需要保证请求头Content-Type为application/json，不然就返回`415 Unsupported Media Type`HTTP状态码。

<span id="pagination"></span>
##  分页

喜欢在返回结果外层嵌套大括号的API通常会在大括号中包含分页数据。我并不会责备它们 - 因为到目前为止，没有更好的方案了。现如今包含详细分页信息的正确做法是使用[RFC 5988提出的Link header](http://tools.ietf.org/html/rfc5988#page-6)。

使用Link header的API可以返回能直接访问的链接，这样API消费者不需要再自己构造链接了。这对于[基于游标](https://developers.facebook.com/docs/reference/api/pagination/)（cursor based）的分页策略尤其重要。这有一个正确使用Link header的例子，从GitHub的文档上扒的：

```
Link: <https://api.github.com/user/repos?page=3&per_page=100>; rel="next",<https://api.github.com/user/repos?page=50&per_page=100>; rel="last"
```

但是这并不是一个万能的方案，因为很多的API会返回额外的分页信息，比如剩余结果的总数。API如果必须要返回这个数量可以使用自定义的HTTP header
`X-Total-Count`。

<span id="autoloading"></span>
##  自动加载相关资源的内容

很多情况下API消费者需要加载跟请求的资源相关（或者相互依赖）的资源。比起为了获取这个信息而对API进行重复请求，在原始资源的响应内容中包含相关资源的数据是一个效率很高的做法。

但是，这么做会[违反一些RESTful原则](http://idbentley.com/blog/2013/03/14/should-restful-apis-include-relationships/)，为了尽可能满足RESTful原则我们可以通过`embed`（或`expand`）查询参数来实现。

在这种情况下，`embed`参数的内容可以是想要被包含的以逗号分隔的字段列表。点操作符用来指明字段的属性。比如：
```
GET /tickets/12?embed=customer.name,assigned_user
```
这个调用将会返回一个包含额外信息的ticket，比如：

```
{
  "id" : 12,
  "subject" : "I have a question!",
  "summary" : "Hi, ....",
  "customer" : {
    "name" : "Bob"
  },
  assigned_user: {
   "id" : 42,
   "name" : "Jim",
  }
}
```
当然，是不是能实现这个功能取决于内部系统的复杂性。这类功能可能会引起[N+1查询问题](http://stackoverflow.com/questions/97197/what-is-the-n1-selects-issue)。

<span id="method-override"></span>
##  覆盖HTTP方法

一些HTTP客户端只支持GET和POST请求。为了能够加强这些客户端的访问能力，API需要一种能够覆盖HTTP方法的做法。尽管这里没有任何强制的标准，但流行的做法是API会接收一个请求头`X-HTTP-Method-Override`，它的值可以是PUT、PATCH或者DELETE三者之一。

注意，用来覆盖HTTP方法的header只能在POST请求中被接受。GET请求永远[不能修改服务器上的数据](http://programmers.stackexchange.com/questions/188860/why-shouldnt-a-get-request-change-data-on-the-server)。

<span id="rate-limiting"></span>

##  请求频次限制

为了防止滥用，给API加上请求频次限制是标准做法。[RFC 6585](http://tools.ietf.org/html/rfc6585)引入了一个HTTP状态码[429 Too Many Requests](http://tools.ietf.org/html/rfc6585#section-4)来解决这个问题。

当然，如果能在API消费者达到调用上限之前能通知到消费者肯定更好。当前这个领域缺少一些标准，但是很多流行的做法是[使用HTTP响应header](http://stackoverflow.com/questions/16022624/examples-of-http-api-rate-limiting-http-response-headers)。

至少得包含以下header（使用Twitter[命名规则](https://dev.twitter.com/docs/rate-limiting/1.1)，header的中间词不需要大写）：

* X-Rate-Limit-Limit - 当前时间段内允许的请求次数
* X-Rate-Limit-Remaining - 当前时间段内剩下的请求次数
* X-Rate-Limit-Reset - 当前时间段还剩下多少秒（也就是还有多久会重置请求限制的次数）

为什么X-Rate-Limit-Reset使用的是剩下的秒数而不是时间戳（timestamp）？

时间戳包含了很多有用但是非必需的信息，比如日期和时区。API消费者真正想知道的是他们什么时候可以继续发起请求，使用秒对于消费者来说处理的成本最小。并且使用秒也避免了[时钟偏移问题](http://en.wikipedia.org/wiki/Clock_skew)

一些API在`X-Rate-Limit-Reset`中使用UNIX时间戳（从1970年1月1日(UTC)到现在的秒数）。别这么做！

为什么说在`X-Rate-Limit-Reset`中使用UNIX时间戳是个坏主意？[HTTP标准](http://www.w3.org/Protocols/rfc2616/rfc2616.txt)中已经[指定](http://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.3)要使用[RFC 1123日期格式](http://www.ietf.org/rfc/rfc1123.txt)（正在[Date](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.18)、[If-Modified-Since](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.25)和[Last-Modified](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.29)等header中使用）。如果我们想要指定一个新的HTTP header，那么应该遵循RFC 1123规定而不是使用UNIX时间戳。

<span id="authentication"></span>
##  认证

RESTful API应该是无状态。这意味着对请求的认证不应该基于cookie或者session。相反，每个请求应该带有一些认证凭证。

如果一直使用SSL，认证凭证可以简单的使用随机生成的access token，把其做为HTTP Basic Auth中user name字段的值传给API。这么做的好处是可以通过浏览器访问 - 如果浏览器从服务器收到`401 Unauthorized`状态码，它将会弹出一个对话框让人输出认证凭证。

当然，这种基于token来进行基本认证的方法只能当用户从API管理后台已经拷贝了一个token到自己的代码中才行。如果搞不到token，只能使用[OAuth 2](http://oauth.net/2/)来把安全token传递给第三方。OAuth 2使用[Bearer token](http://tools.ietf.org/html/rfc6750)，并且也是基于SSL来保证传输安全。

支持JSONP的API可能需要第三种方法来实现认证，因为JSONP的请求没法发送HTTP Basic Auth凭证或者Bearer token。这种情况下，可以使用一个额外的查询参数`access_token`。注意：使用查询参数来传递token存在一个固有的安全隐患，因为大多数web服务器会在服务器日志中保存查询参数。

不管怎么样，以上三种方法是用来在API之间传输token的方法。实际传输的token可以是一样的。

<span id="caching"></span>
##  缓存

HTTP内置了缓存策略。你只需要在API响应中增加几个header，在处理请求的时候对一些请求header做点校验。

这有两个方案：[ETag](http://en.wikipedia.org/wiki/HTTP_ETag)和[Last-Modified](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.29)

ETag：当处理一个请求的时候，在响应中包含一个名为`ETag`的HTTP header，它的值可以为资源内容的hash或者checksum。ETag的值应该在资源内容发生变化的时候跟着变化。如果HTTP请求中包含`If-None-Match`header，并且这个header的值与被请求资源的ETag值相同，那么API应该返回`304 Not Modified`状态码，而不是输出资源内容。

Last-Modified：基本上跟ETag的工作原理差不多，区别在于这个header使用的是时间戳。响应header`Last-Modified`中包含一个[RFC 1123](http://www.ietf.org/rfc/rfc1123.txt)格式的时间戳，用来对`If-Modified-Since`的值进行校验。注意HTTP协议接受[三种不同的日期格式](http://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.3)，所以对这三种格式服务器应该都能处理。

<span id="errors"></span>
##  错误处理

就像HTML的出错页面向访问者展示了有用的错误消息一样，API也应该用之前熟悉易读的格式来提供有用的错误消息。错误的表现形式应该跟其他资源保持一致，只是用一些自己的字段。

API应该一直返回合理的HTTP状态码。API错误一般情况下分成两类：代表客户端错误的400系列状态码和代表服务端错误的500系列状态码。API至少把所有400系列错误统一用易读的JSON格式来展示。如果可能（比如，如果负载均衡和反向代理能够创建自定义错误内容的话），500系列的状态码也这么弄。

JSON错误内容应该为开发者提供一些东西 - 有用的错误消息，唯一的错误码（通过它可以在文档中找到更多错误细节），可能的话提供错误细节描述。用JSON格式来输出错误看起来这样：

```
{
  "code" : 1234,
  "message" : "Something bad happened :(",
  "description" : "More details about the error here"
}
```

对于PUT、PATCH和POST的请求进行的校验错误需要嵌套多个字段。最佳做法是用固定的错误码来表示校验失败，然后在额外的`errors`字段中提供错误的细节，像这样：

```
{
  "code" : 1024,
  "message" : "Validation Failed",
  "errors" : [
    {
      "code" : 5432,
      "field" : "first_name",
      "message" : "First name cannot have fancy characters"
    },
    {
       "code" : 5622,
       "field" : "password",
       "message" : "Password cannot be blank"
    }
  ]
}
```

<span id="http-status"></span>
## HTTP状态码

HTTP定义了很多[有意义的状态码](http://en.wikipedia.org/wiki/List_of_HTTP_status_codes)，你可以在你的API中使用。这些状态码可以帮助API消费者用来路由它们获取到的响应内容。我整理了一个你肯定会用到的状态码列表：

* 200 OK - 对成功的GET、PUT、PATCH或DELETE操作进行响应。也可以被用在不创建新资源的POST操作上
* 201 Created - 对创建新资源的POST操作进行响应。应该带着指向新资源地址的[Location header](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.30)
* 204 No Content - 对不会返回响应体的成功请求进行响应（比如DELETE请求）
* 304 Not Modified - HTTP缓存header生效的时候用
* 400 Bad Request - 请求异常，比如请求中的body无法解析
* 401 Unauthorized - 没有进行认证或者认证非法。当API通过浏览器访问的时候，可以用来弹出一个认证对话框
* 403 Forbidden - 当认证成功，但是认证过的用户没有访问资源的权限
* 404 Not Found - 当一个不存在的资源被请求
* 405 Method Not Allowed - 所请求的HTTP方法不允许当前认证用户访问
* 410 Gone - 表示当前请求的资源不再可用。当调用老版本API的时候很有用
* 415 Unsupported Media Type - 如果请求中的内容类型是错误的
* 422 Unprocessable Entity - 用来表示校验错误
* 429 Too Many Requests - 由于请求频次达到上限而被拒绝访问
