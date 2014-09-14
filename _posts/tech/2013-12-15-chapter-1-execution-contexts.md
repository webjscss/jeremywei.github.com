---
layout: post
title: 「深入ECMA-262-3」第一章、执行上下文
city: 南京
tags: [tech, translate]
---

原文：[http://dmitrysoshnikov.com/ecmascript/chapter-1-execution-contexts/](http://dmitrysoshnikov.com/ecmascript/chapter-1-execution-contexts/)

1. [介绍](#introduction)
2. [定义](#definitions)
3. [可执行代码的类型](#types-of-executable-code)
	1. [全局代码](#global-code)
	2. [函数代码](#function-code)
	3. [Eval代码](#evalcode-code)
4. [总结](#conclusion)
5. [额外资料](#additional-literature)

<span id="introduction"></span>
#介绍

在这篇文章中，我们将会谈论到ECMAScript的执行上下文（execution contexts）以及与其相关的可执行代码（executable code）的类型。

<span id="definitions"></span>
#定义

每次当控制权转移到ECMAScript可执行的代码中的时候，控制权也进入了一个_执行上下文中_。

	执行上下文（缩写－EC）是ECMA-262规范用来区分可执行代码而使用的一个抽象概念。


这个标准没有从技术实现的角度为EC定义准确的结构和种类；对于ECMAScript引擎来说实现这个标准是个问题。

从逻辑上来说，一些活动的执行上下文会组成一个栈。栈底一直是一个_全局上下文_，栈顶－是当前（活动）执行上下文。在进入和退出各种EC的时候，这个栈会被修改（push/pop）。

<span id="types-of-executable-code"></span>
#可执行代码的类型

由于执行上下文的抽象概念，_可执行代码的类型(type of an executable code)_的概念被引入。说到代码类型，它在一定的时刻可以意指执行上下文。

比如，我们把执行上下文栈定义为一个数组：
	
	ECStack = [];

在每次在进入一个函数（即使这个函数被递归的调用或者被当作构造函数），同时也包括内置的eval函数的时候，这个栈会被推入相应的元素。

<span id="global-code"></span>
##全局代码

这个类型的代码是在等级```程序```（level Program）中进行被执行：换句话说，也就是加载了的外部```.js```文件或者本地的行内代码（位于```<script></script>```标签之中）。全局代码不包含任何位于函数中的代码。

在初始化阶段（程序启动），```ECStack```看起来是这样的：

	ECStack = [
	  globalContext
	];

<span id="function-code"></span>
##函数代码

在进入函数代码（所有类型的函数）的时候，```ECStack```会被推入新的元素。很有必要指出一下具体函数的代码不包括内部函数的代码。

比如，让我们看看这个递归调用自身一次的函数：

	(function foo(flag) {
	  if (flag) {
	    return;
	  }
	  foo(true);
	})(false);
	
然后，ECStack被如下修改:

	// first activation of foo
	ECStack = [
	  <foo> functionContext
	  globalContext
	];
  
	// recursive activation of foo
	ECStack = [
	  <foo> functionContext – recursively 
	  <foo> functionContext
	  globalContext
	];

函数的每次返回都会使程序从当前的执行上下文中退出，并且```ECStack```也相应的弹出对应的元素－如此反复－一个典型的栈的行为。在这段代码的工作完成之后，```ECStack```又会只包含```全局上下文```－直到程序结束为止。

一个抛出的但是没有被捕获的异常也可以退出一个或多个执行上下文：

	(function foo() {
	  (function bar() {
	    throw 'Exit from bar and foo contexts';
	  })();
	})();

<span id="evalcode-code"></span>
##Eval代码

与```eval```代码相关的东西更加有趣。在这个例子中，存在一个```调用上下文(calling context)```的概念，换句话说就是```eval```函数被_调用_时所在的上下文。

```eval```所进行的操作，比如像变量或者函数定义，会确切的影响_调用_上下文：

	// influence global context
	eval('var x = 10');
 
	(function foo() {
	  // and here, variable "y" is
	  // created in the local context
	  // of "foo" function
	  eval('var y = 20');
	})();
  
	alert(x); // 10
	alert(y); // "y" is not defined
	
注意，在ES5的[严格模式（strict-mode）](http://dmitrysoshnikov.com/ecmascript/es5-chapter-2-strict-mode/)中，```eval```已经_不会_影响调用上下文了，但是会影响在本地_沙盒(sandbox)_中的代码。

对于以上的例子，ECStack将会有如下变化：

	ECStack = [
	  globalContext
	];
  
	// eval('var x = 10');
	ECStack.push({
	  context: evalContext,
	  callingContext: globalContext
	});
 
	// eval exited context
	ECStack.pop();
 
	// foo funciton call
	ECStack.push(<foo> functionContext);
 
	// eval('var y = 20');
	ECStack.push({
	  context: evalContext,
	  callingContext: <foo> functionContext
	});
 
	// return from eval 
	ECStack.pop();
 
	// return from foo
	ECStack.pop();

也就是一个非正式并且有逻辑性的调用栈。

在旧的SpiderMonkey实现中（Firefox），到版本1.7为止，可以传递一个调用上下文做为_第二个_参数给```eval```函数。因此，如果这个上下文仍然存在，那么就可以对私有变量产生影响：

	function foo() {
	  var x = 1;
	  return function () { alert(x); };
	};
 
	var bar = foo();
 
	bar(); // 1
 
	eval('x = 2', bar); // pass context, influence internal var "x"
 
	bar(); // 2
	However, due to security reasons in modern engines it was fixed and is not significant anymore.

<span id="conclusion"></span>	
#总结

这个简化了的理论是以后分析与执行上下文相关内容细节所必需的，比如[像变量对象( variable object )](http://dmitrysoshnikov.com/ecmascript/chapter-2-variable-object/)或者[作用域链(scope chain, )](http://dmitrysoshnikov.com/ecmascript/chapter-4-scope-chain/)，关于它们的描述可以在对应章节找到。

<span id="additional-literature"></span>
#额外资料

ECMA-262-3规范的对应部分－[10. Execution Contexts](http://bclary.com/2004/11/07/#a-10)



