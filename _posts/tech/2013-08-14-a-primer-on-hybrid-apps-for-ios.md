---
layout: post
title: iOS混合应用开发入门
city: 南京
tags: [translate,tech]
---

原文：[https://www.cocoacontrols.com/posts/a-primer-on-hybrid-apps-for-ios](https://www.cocoacontrols.com/posts/a-primer-on-hybrid-apps-for-ios)

#介绍

上周（译者：原文成于2012.07.06），[纽约时报透露说Facebook正在致力于对其iOS应用进行重大升级](http://bits.blogs.nytimes.com/2012/06/27/facebook-plans-to-speedup-its-iphone-app/?_r=0)。这件事本身没有什么新闻价值。Facebook当然在致力于对其iOS应用进行重大升级。但是，这次特别的升级相当有新闻价值。就如何构建和维护越来越多的移动应用套件而言，Facebook正在计划一个意义重大的航线修正（译者：技术转型）。

到目前为止，Facebook公开的移动策略是为了避免「重复写四次相同代码」（之后会更多），会发布混合应用（hybrid　apps）到各主流平台。理论上讲，这个想法很完美：当你可以为相同的功能只维护一个代码库的时候，谁愿意为此维护多个呢？但是，实际的情况是，Facebook的应用在任何平台上都没有「原生」的感觉，同时也被严重的性能问题所折磨，并普遍的被其用户所辱骂。

这篇文章将会对混合应用的主题进行更深入的探讨。我们将会以讲解混合应用的构成做为开始。然后，我们将会看下混合应用所能提供的益处，并且会提供更多消极方面的内容细节。我们将会检验一些可以使混合应用在感觉上更加像一个原生应用的选项，然后给你提供一些用来优化性能，外观以及体验的建议。最后，我们会把注意力转回到Facebook上，为了理解他们是如何成为今天这个样子的，我们将会更细致的查看他们iOS应用的历史。

从个人的角度来说，我不会为iOS构建一个混合应用，除非我必须这么做不可。我不认为混合应用在iOS上表现良好：它们比较慢，也比较笨拙，并且从来没有原生应用那样的外观和体验。但是，在合适的场景下，它们所能提供的优势能抵掉它们的劣势。为了学习关于混合应用的所有内容，什么是能做的，什么是不能做的，并且它们是否满足你的需求，请继续往下读。

# 什么是混合应用？

通常来说，一个iPhone应用是利用Cocoa Touch框架以纯Objective C来构建的。你可能会有一个`UITabBarController`，上面安放了一些视图控制器（view controllers）。这些视图控制器可能是`UITableViewController`的子类，或者也有可能是`UIViewController`，其UI是使用`XIB`所定义或者在代码中定义。有时候，一个视图控制器上可能会安放一个`UIWebView`控件，用来在一个应用的内部展示web内容或长文本内容。

混合应用用HTML，CSS和Javascript而不是Objective C来实现部分或者所有的客户端代码。一个特定应用的屏幕实际上可能包含一个`UIWebView`，用它来渲染服务端返回的标记语言（译者：即HTML）并且解释服务端返回的代码（译者：JavaScript）而不是Objective C。

一些应用，比如像Instapaper和Pocket，是高度依赖web视图来实现其关键的应用功能的。Instapaper和Pocket对web视图的使用采取了一个务实的方案：两个应用的主函数都是展示重新格式化的web内容，所以只用web视图来展示HTML是有道理的。

就像在下面的类型列表中即将看到的那样，Facebook尽可能的在它们的iOS应用中使用HTML，其本质上是把[http://touch.facebook.com](http://touch.facebook.com)的一个版本填充进了一个Objective C的壳里边。它们在一些限定的应用特性中使用Objective C，比如登录窗口和它们曾经独一无二的滑出菜单界面。

最后，还存在可以添加到iPhone主屏的移动站点。这些站点是100%用HTML构建的，并且特定的运行在移动版Safari中。

#应用的类型

<table class="table table-bordered table-striped table-condensed">
    <caption>应用类型</caption>
    <thead>
        <tr>
            <th scope="col">类型</th>
            <th scope="col">例子</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>完全原生</td>
            <td>Path 2.0和其他完全利用Cocoa Touch框架用Objective C所实现的应用。</td>
        </tr>
        <tr>
            <td>一些web视图</td> <td>Instapaper和Pocket都是使用的web视图来实现的关键的应用功能，但是其他的功能是用Objective C来实现的。</td>
        </tr>
        <tr>
            <td>最小化Objective C</td> 
			<td>比如Facebook。尽可能的使用HTML/JS/CSS来实现的应用，并且只有很少的组件是原生的。</td>
        </tr>
        <tr>
            <td>纯HTML</td> 
			<td>就像它描述的那样</td>
        </tr>
    </tbody>
</table>

#混合应用的优势

为什么你想要创建一个混合应用？这儿有一些原因，比如产品开发和更新的速度，轻松的进行A/B测试，以及通过让前端web开发人员为你的iOS应用贡献力量，可以增加可用的开发人员数量。

##开发速度

尽管开发一个外观完美，行为原生的移动web体验是不太可能一夜完成的，但是在较短的时间内开发一个合宜的体验相对来说是比较简单的。开发一个得体的web体验，并且把它加载到一个`UIWebView`中肯定比开发一个JSON API，并用Objective C来写一些前端代码要容易。由于这个原因，一些开发者可能比较喜欢用HTML来交付那些用来检验想法的功能，然后接下来当功能的价值被证实了的时候再把功能的代码替换成原生代码。

投入市场的时间是公司之间的关键区分点，发布日期延迟几周或者甚至几个月对于资源受限（（[limited runway](http://pollenizer.com/runway-thinking-a-practical-guide-to-startup-survival/)）的创业公司来说这可能意味着生与死的区别。

##发布更新的速度

通过HTML来提供产品功能的另一个优势是你可以在不向苹果发起应用更新请求的情况下去更新你应用的功能集合，并且不需要等待苹果几周的审批过程。快速的进行改变，修复问题以及能扩展你应用的功能，这可以为你提供一个用来超越对手的重要竞争优势，你的竞争对手由于每个小的改动都需要经过苹果的审批过程，所以它们的灵活性就比较低。

早上想要的新功能，这天结束的时候就可以发布，并且晚上就可以观测用户的使用情况，以便第二天早上就可以对其进行优化，调整或者删除，这是对体验进行打磨非常强大的方法，而体验恰恰是你的应用可以在市场上取得成功的必需条件。

##A/B测试

与开发的速度和发布更新的速度都相关的是快速的对用户进行[A/B测试或者分组测试](http://en.wikipedia.org/wiki/A/B_testing)的能力。比如，你可能想测试一下你应用上的新feed功能，在其自身包含一个`Like`按钮的情况下和只在feed内容详情页面包含这个按钮的情况，用户使用率是否有所提高。用HTML来实现feed功能，你可以很容易的对这个更改进行分开测试，为了从统计数据上来看这个重要的用户行为是否发生，可以只在一组用户中展示这个按钮。

在原生代码中执行A/B测试不是不可能的，但是准备测试用例会更加具有挑战性，并且对比起一个混合应用来说，从测试用例中返回对应的新数据这个过程将会显著地耗费更多的时间。如果你对如何在Objective C中实现A/B测试好奇，[Little Big Thinkers上有一篇好文章展示了它们如何在iPhone上进行A/B测试](http://littlebigthinkers.com/post/how-to-run-ab-tests-in-ios-apps)（译者：翻墙）。

##App开发的民主方式

如果你曾经尝试过雇佣一个iOS开发者（或者如果你是一个iOS承包商），你知道对靠谱iOS开发者的需求与可雇佣的人来说早就已经是供不应求。混合应用可以大大的增加你所需要的靠谱开发者数量，因为你可以让你任何的前端开发人员转做你iOS应用中的功能开发，假设他们已经熟悉Javascript，HTML，CSS和你服务器上使用的任何后端语言。

#混合应用的劣势

当然，如果混合应用没有缺点，没人想开发原生应用。现实中显然不是这种情况，并且这里有几个原因解释了为什么会这样。概括来说，混合应用较慢，较臃肿，看起来不像原生的感觉，并且，最重要的是，没有原生的体验。

##Nitro的缺失

Nitro是移动Safari的Javascript引擎，[速度太TMD快了](http://www.guypo.com/mobile/ios5-top10-performance-changes/)。不幸的是，由于安全的关系，[UIWebViews无法享受到这个速度提升带来的优势](http://daringfireball.net/2011/03/nitro_ios_43)：

_Nitro比WebKit之前的JavaScript引擎性能有所提升的最大原因是其采用了JIT－"Just-In-Time" 编译... JIT需要可以把RAM中的内存页标记为可执行状态的能力，但是，iOS，出于安全的考量，不允许内存页被标记为可执行状态。这是一个重要并且严格的安全策略。大部分现代操作系统允许内存页被标记为可执行状态－包括Mac OS X，Windows，和（我相信）Android。iOS 4.3对这个策略有个例外，但是这个例外只限于移动Safari。_

实际上来说，这意味着如果你的混合应用使用了Javascript，那么你将会感觉到同样的UI要比在移动Safari中要慢，也比以Objective C实现的同样的UI要慢。不幸的是，这儿没有简单的方法来解决这个问题，除了减少或者全部移除UIWebViews中你所使用的Javascript之外，尽可能的以CSS3动画（animations）和过渡（transitions）来做为替代（这可以利用GPU加速的优势）。

![css3](http://{{ site.cdn }}/images/hybrid-ios/css3.jpg)

为了弄清楚，你可以只使用CSS3来完成一些真实的不可思议的效果，[就像刚才Hakim El Hattab在他的stroll.js项目所演示的那样](http://lab.hakim.se/scroll-effects/)。但是，总的来说，Nitro的缺失是任何混合iOS应用成功的严重障碍，特别是如果想获得与原生iOS应用同样的外观和体验。

##模仿原生UI的挑战

除了上面描述的性能问题之外，模仿原生Cocoa Touch用户界面的挑战是实现一个混合应用所面对的另一个困难。就像之前被指出过无数次，iPhone应用拥有非常不同寻常的外观和体验。比如，Table cell拥有标准的字体大小，内边距和空白要求，gradient　selection高亮，disclosure箭头，以及许多其他很难被复制的特性。

一个替代的方式是放弃对iOS系统控件的复制，相反为你的应用构建一个独一无二的UI，就像Facebook的News Feed和Timeline功能那样。但是，即使你选择了这个方法，为了使你的混合应用在感觉上不像一个web页面，这儿还有一些WebKit特性需要被解决。

##有时候事情很容易变糟

预计迟早有一天，你混合视图中的CSS或者Javascript文件将会加载失败，这不是没有理由的。可能你的用户是纽约或者旧金山的AT&T的客户，众所周知他们使用的是不稳定的GSM网络。可能上帝今天仅仅是没对你笑而已。不管怎么样，终有一天你将会给用户展示一个像这样的UI：

![fb_broken](http://{{ site.cdn }}/images/hybrid-ios/fb_broken.jpg)

…并且最糟糕的是你对于这种情况几乎做不了任何事情，除了把你的JS和CSS都混入HTML页面中。这种情况对于你和你的应用都是极为糟糕的，并且它是完全无法控制的。至少对一个完整的原生应用来说，服务端错误可以通过一个弹出的警告来更加优雅的进行处理。
（截屏来自[@timanrebel](https://twitter.com/timanrebel)，[社会化滑雪和滑雪板应用，Snowciety](http://snowcietyapp.com/)的创始人。）


#如何使你的混合应用拥有原生的体验

##UIWebView的阴影和背景颜色

默认情况下，`UIWebViews`会显示一个阴影，以及背景颜色或者一个位于渲染页面下面的图案（取决于iOS版本和硬件）。这个外观和感觉明显是非原生的，所以你将会需要移除它们两个。幸运的是，这么做很简单：为了使视图变得扁平（flat），所有你需要做的事情就是一个`UIWebView`子类里边的几行代码。check out我的那个MIT协议的项目，[FlatWebView](http://cocoacontrols.com/platforms/ios/controls/flatwebview)，看看例子里边是如何做的。

![webview](http://{{ site.cdn }}/images/hybrid-ios/webview.jpg)


##链接的高亮显示

为了帮助用户知道他们究竟触碰过哪里，当链接被触碰的时候WebKit会在其周围高亮的显示一个半透明的矩形。

![highlight](http://{{ site.cdn }}/images/hybrid-ios/highlight.jpg)

尽管这对于web来说很便捷，但是它在感觉上明显不是原生的。为了清除这个行为，你可以在你的web应用中包含这个CSS片段：

	/*http://stackoverflow.com/questions/9157080/wrong-webkit-tap-highlight-color-behavior-when-page-as-web-standalone-app */ 

	html { 
		-webkit-tap-highlight-color: rgba(0,0,0,0); 
		-webkit-user-select: none; 
	}

##触摸延迟

默认情况下，当WebKit对web页面上的被触摸的链接进行响应的时候，它会增加一个大约300毫秒的延迟。这实际上是一个特性，不是一个bug，由于意外的触碰一个web页面上的错误链接是特别常见的，所以拥有一个细微的延迟可以给用户提供修正他们错误的机会，对于什么都不做来说这可以创造更好的体验。无论如何，当你正在创建一个混合应用的时候，你应该只使用触碰目标尺寸大得可以避免被意外触碰的UI组件。（你正在使用最小尺寸为40×40px的大尺寸触碰目标，对吗？）

为了那个目的，当你在创建一个混合应用的时候你必须禁用WebKit的触摸延迟的特性，确保你应用的UI在感觉上和Cocoa Touch的UI拥有一样的响应体验。Matteo Spinelli提供了一些方便的Javascript代码（不是jQuery！）用来解决这个「问题」：[Remove onclick delay on Webkit for iPhone.](http://cubiq.org/remove-onclick-delay-on-webkit-for-iphone)

##滚动

过去创建一个混合应用最大的挑战之一就是在一个UIWebView中复制原生的滚动行为。特别是，在`UIWebView`中的`UIScrollView`的子类几乎不可能去复制滚动速度，惯性以及「橡皮圈」（rubberbanding）视觉效果。幸运的是，苹果在iOS 5中为`UIWebView`增加了一个`scrollView`属性，这使得对上面这些功能的支持是小菜一碟。

你所需要做的就是把你web视图上滚动视图的`decelerationRate`属性设置为`UIScrollViewDecelerationRateNormal`：

	UIWebView *hybridView = [self somethingThatGetsOurWebView];
	webView.scrollView.decelerationRate = UIScrollViewDecelerationRateNormal;


##Objective C与Javascript的桥接

如果你不想利用一些iOS平台能力的优势，那么创建一个混合应用就没有什么意义。比如，即使当前Facebook大部分是用HTML实现的，但是它仍然在使用原生的UI，包括在其他用户的时间轴上留言，上传图片，下拉刷新，以及其他功能。很明显，你应用中的原生功能必须可以与你`UIWebView`上的视图进行交互。在原生功能与web视图之间创建一个无缝的体验是很具有挑战性的，虽然这样，还是要给出这些层次之间存在的简陋交互方式。

###Objective C到Javascript

为了在原生代码发生改变的时候去更新一个web视图，你拥有三个选项：

*  重新加载UIWebView的内容。
*  使用NSURLRequest加载一个新页面。
*  在你的UIWebView中使用-stringByEvaluatingJavaScriptFromString:这个API来执行Javascript代码。

重新加载@UIWebView@中的内容－或者加载一个全新的页面－这糟透了，因为这将会在一个无法忽视的时间内展示给你一个空白，无法操作的页面。你可以一直显示一个loading HUD视图（比如[SVProgressHUD](http://cocoacontrols.com/controls/svprogresshud)），但是这仍然不是一个非常好的体验。

在UIWebView中执行Javascript代码来更新视图是一个较好的选项：比如，这给你提供了执行部分更新的机会。不过，它也有它自己的挑战，比如极其糟糕的调试体验，当一些功能不能正常工作的时候你就会感受到了。

###Javascript到Objective C

为了在当你的web视图发生改变的时候去更新原生代码，与iOS打赌（译者：交互）的最佳方式是实现`UIWebView`的委托方法－webView:shouldStartLoadWithRequest:navigationType:。为了可以从`UIWebView`中调用原生平台的特性，你可以定义自己的scheme（比如用`myappname://`代替`http://`）和自己的URIs（比如用`showImageCapture`代替apple.com）。不幸的是，这个方案即使对于复杂程度不高的应用来说也会退化为一个庞大的if区块。

比如说Facebook可能像这样来实现他们的委托方法：

	if ([request.url.scheme isEqual:@"showImagePicker"]) { 
		// show a UIImagePickerController 
	} else if ([request.url.scheme isEqual:@"sendMessage"]) { 
		// show the send message view controller 
	} else if ([request.url.scheme isEqual:@"checkIn"]) { 
		// show the places checkin view controller 
	} else if ([request.url.scheme isEqual:@"postStatus"]) { 
		// show the post status view controller 
	}

幸运的是，这里有一些方法可以简化它。比如，你可以在某种点对点的调度系统中使用一个`NSDictionary`来对scheme到action进行路由。

	- (void)viewDidLoad
	{
	    self.dispatch = [NSMutableDictionary dictionary];
	    [dispatch setObject:[NSValue valueWithPointer:@selector(showImagePickerForURL:)] forKey:@"showImagePicker"];
	}

	- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType
	{
	    NSValue *selectorPointer = [self.dispatch objectForKey:request.url.scheme];
	    if (selectorPointer)
	    {
	      [self performSelector:[selectorPointer pointerValue] withObject:request.url];
	      return NO;
	    }
	    else
	    {
	      return YES;
	    }
	}
	
#Facebook

当Facebook的iPhone应用首次亮相的时候，它几乎是个完整的原生应用，是由Joe Hewitt单枪匹马写出来的（据我所知），他把[Three20 iOS框架](http://three20.info/)从应用的核心中提取了出来。如果你曾经用Three20开发过应用，你知道这个框架有一点笨拙。可能，在[Joe退出iOS开发团队](http://techcrunch.com/2009/11/11/joe-hewitt-developer-of-facebooks-massively-popular-iphone-app-quits-the-project/)之后的一段时间，Facebook的领导们判断出当前应用的实现简直是太难维护了，是时候重新考虑方案了。

Dave Fetterman，Facebook平台的工程经理，他在去年F8大会一个演讲中的[部分篇幅描述了这个突变](http://www.readwriteweb.com/mobile/2011/09/how-facebook-mobile-was-design.php)：

_由于一些根本原因你需要为四个不同的平台开发应用。你想为所有的这些平台各自开发？那你将会不得不像SB一样开发四次。然后这是所有的特性－群组，交易，新的profile。这么说来，我们不得不开发四次，这意味着开发的速度变慢了。代码变得陈旧了。这里会存在不能一起工作的不同版本的产品和事情，这对于像Facebook这样快速发展的公司来说极其困难。_

[所以，当Facebook4.0的iOS版本发布的时候](http://www.insidefacebook.com/2011/10/10/facebook-for-iphone-4-0-ipad/)，它体现出了"Write Once, Run Anywhere"的思想。就个人来讲，我必须说，我认为在理论上这是一个伟大的想法。没人想开发和维护相同的功能集合四次，特别是像Facebook这样的公司，它拥有惊人的高「用户－工程师」比例。很不幸，真实的情况是`UIWebView`并没有足够的能力来处理像Facebook这样的富应用的需求，用户的眼睛是雪亮的：

![reviews](http://{{ site.cdn }}/images/hybrid-ios/reviews.png)

#总结

综上所述，混合应用在iOS上的体验不太好：它们较慢，也较笨拙，可能引起一些无法修复的错误，并且它们在感觉上并不能和原生应用相比。但是，它们所提供一些优势在特殊的境况之下，可能是一个有价值的折中方案。个人来讲，我建议你应该一直用原生代码来开发你整个iOS应用的用户界面，然后根据需求所要求的那样有选择的把应用中的部分功能改造成混合方式，不管需求是纯开发速度，即刻更新的能力，或者在很短的时间内在用户身上去测试功能的多个变化。

最后，无论怎样，对你的用户和你的业务来说，只有你知道什么才是真正正确的。保证你是因为正确的原因而做的选择，而不是纯粹的图方便。

对于混合应用你是怎么看的？什么时候你觉得利大于弊？你还有其他可以提高混合应用的体验的窍门和技巧吗？还需要更多如何从混合开发转向完全原生开发的建议？请留言，我们期望听到你对这个问题的声音。