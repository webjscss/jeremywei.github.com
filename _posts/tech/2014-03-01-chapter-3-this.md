---
layout: post
title: 「深入ECMA-262-3」第三章、This
city: 南京
tags: [tech, translate]
---

原文：[http://dmitrysoshnikov.com/ecmascript/chapter-3-this/](http://dmitrysoshnikov.com/ecmascript/chapter-3-this/)

1. [介绍](#introduction)
2. [声明](#definitions)
3. [全局代码中的this值](#this-value-in-the-global-code)
3. [函数代码中的this值](#this-value-in-the-function-code)
	1. [引用类型](#-reference-type)
	2. [函数调用和非引用类型](#function-call-and-non-reference-type)
	3. [引用类型和this的null值](#-reference-type-and-null-this-value)
	4. [构造函数中this的值](#this-value-in-function-called-as-the-constructor)
	5. [调用函数时手动设置this值](#manual-setting-of-this-value-for-a-function-call)
4. [总结](#conclusion)
5. [额外资料](#-additional-literature)

<span id="introduction"></span>
## 介绍

在这篇文章中我们将会再次讨论和[执行上下文](/chapter-1-execution-contexts.html)直接相关的细节。讨论的主题是```this```关键字。

就像例子展示的那样，这个主题较难并且在不同执行上下文中确定```this```值的时候经常会引起问题。

许多程序员过去一直认为编程语言中的```this```关键字是和面向对象编程（OOP）紧密相关的，确切的说就是引用构造方法刚创建的对象。在ECMAScript中这个概念也被实现了，但是，就像我们将要看到的，在这里它不仅限于去引用创建过的对象。

让我们详细看下ECMAScript中的this值到底为何物。

<span id="definitions"></span>
## 声明

`this`的值是执行上下文中的一个属性：

	activeExecutionContext = {
	  VO: {...},
	  this: thisValue
	};

其中VO是我们在之前章节中讨论过的[变量对象](/chapter-2-variable-object/)。

`this`是和上下文中的[可执行代码类型](/chapter-1-execution-contexts.html#types-of-executable-code)紧密相关的。它的值在进入上下文的时候被决定，并且当代码在上下文中运行的时候是不可变的。

让我们更加详细地看下这些情况。

<span id="this-value-in-the-global-code"></span>
## 全局代码中的this值

事情非常简单。在全局代码中，`this`的值一直是全局对象本身。因此，可以间接的引用它：

	// explicit property definition of
	// the global object
	this.a = 10; // global.a = 10
	alert(a); // 10

	// implicit definition via assigning
	// to unqualified identifier
	b = 20;
	alert(this.b); // 20

	// also implicit via variable declaration
	// because variable object of the global context
	// is the global object itself
	var c = 30;
	alert(this.c); // 30

<span id="this-value-in-the-function-code"></span>
## 函数代码中的this值

当在函数代码中使用```this```的时候，事情变得更加有趣。这个情况是最难的，并且会引起很多问题。

在这种类型代码（函数代码）中，```this```值的第一个（可能是主要的）特性是它并不是静态得和函数绑定。

像上面所提到的，```this```的值是在进入上下文的时候所决定的，并且如果是函数代码的话，它的值可能每次都不一样。

同时，在代码运行时```this```的值是不能更改的，换句话说，不能对它进行赋值，因为它不是变量（与Python编程语言相比，它显示的定义了```self```对象，并且可以在运行时多次进行修改）：

	var foo = {x: 10};

	var bar = {
	  x: 20,
	  test: function () {

	    alert(this === bar); // true
	    alert(this.x); // 20

	    this = foo; // error, can't change this value

	    alert(this.x); // if there wasn't an error, then would be 10, not 20

	  }

	};

	// on entering the context this value is
	// determined as "bar" object; why so - will
	// be discussed below in detail

	bar.test(); // true, 20

	foo.test = bar.test;

	// however here this value will now refer
	// to "foo" – even though we're calling the same function

	foo.test(); // false, 10

那么是什么引起了函数代码中```this```值的变化？这有几个方面的原因。

首先，在普通的函数调用中，```this```的值是由激活了上下文中代码的调用者所提供，也就是调用这个函数的父作用域。```this```的值是由调用表达式（call expression ）的形式（换句话说是函数调用的语法形式）所决定的。

为了能够在任何上下文中准确无误地确定```this```的值，理解和记住这个关键点是非常必要的。确切的说，调用表达式的形式，也就是调用函数的形式，影响了被调用的上下文中```this```的值，这就是全部。

（就像我们在一些文章甚至一些关于JavaScript的图书中都声称「```this```的值取决于函数定义的方式：如果是全局函数，那么```this```的值被设置为全局对象，如果函数是一个对象的方法，那么```this```的值会一直是这个对象」－这是错误的表述）。往下看，我们将会看到即使是普通的全局函数也可以通过不同形式的调用表达式进行调用，从而影响```this```的值：

	function foo() {
	  alert(this);
	}

	foo(); // global

	alert(foo === foo.prototype.constructor); // true

	// but with another form of the call expression
	// of the same function, this value is different

	foo.prototype.constructor(); // foo.prototype

类似地，我们可以调用某个对象的方法，但是```this```的值不会被设置为这个对象：

	var foo = {
	  bar: function () {
	    alert(this);
	    alert(this === foo);
	  }
	};

	foo.bar(); // foo, true

	var exampleFunc = foo.bar;

	alert(exampleFunc === foo.bar); // true

	// again with another form of the call expression
	// of the same function, we have different this value

	exampleFunc(); // global, false

那么调用表达式的形式是如何影响```this```值的？为了能够全面理解```this```值的确定过程，那么必须详细的了解内部类型之一－```引用```(Reference)类型。

<span id="-reference-type"></span>
###  引用类型

如果使用伪代码，那么引用类型的值可以表示为一个对象，此对象拥有两个属性：base（也就是属性的所有者）以及这个base中的属性名称（propertyName）：

	var valueOfReferenceType = {
	  base: <base object>,
	  propertyName: <property name>
	};


`引用`类型的值只在两种情况下会存在：

1. 当我们处理标识符；
2. 或者属性访问器（property accessor）的时候。

标识符是在标识符确定的过程中进行处理的，这个过程在[第四章、作用域链](http://dmitrysoshnikov.com/ecmascript/chapter-4-scope-chain/)进行了详细说明。并且我们注意到在那里这个算法一直会返回一个引用类型的值（这对于```this```的值很重要）。

标识符包括变量名，函数名，函数参数名以及全局对象属性名。举个例子，对于如下标识符的值：

	var foo = 10;
	function bar() {}

在执行的中间结果中，相应的引用类型的值为：

	var fooReference = {
	  base: global,
	  propertyName: 'foo'
	};

	var barReference = {
	  base: global,
	  propertyName: 'bar'
	};

为了能够从```引用```类型的值中获取一个对象真实的值，所以存在一个名为```GetValue```的方法，其用伪代码描述如下：

	function GetValue(value) {

	  if (Type(value) != Reference) {
	    return value;
	  }

	  var base = GetBase(value);

	  if (base === null) {
	    throw new ReferenceError;
	  }

	  return base.[[Get]](GetPropertyName(value));

	}

这个名为```[[Get]]```的内部方法会返回对象属性真实的值，也会解析从原型链（prototype chain）中继承的属性：

	GetValue(fooReference); // 10
	GetValue(barReference); // function object "bar"

属性访问器也是；这里有两种形式：点标记法（当属性名是正确的标识符，并且是可提前预知的时候使用），或者括号标记法：

	foo.bar();
	foo['bar']();

在中间计算返回的时候，我们也会得到```引用```类型的值：

	var fooBarReference = {
	  base: foo,
	  propertyName: 'bar'
	};

	GetValue(fooBarReference); // function object "bar"

那么，```引用```类型的值和函数上下文的```this```值是如何联系到一起的？－从最重要的意义上来说。下面是本文的核心。在函数上下文中确定```this```值的基本规则听起来是这样的：

函数上下文中```this```的值是由调用者所提供的，并且是由当时的调用表达式的形式（函数调用的语法是如何写的）来决定的。

如果调用圆括号( ... )的左侧是一个```引用```类型的值，那么```this```的值被设置为这个```引用```类型值的base对象。

其他情况下（也就是除```引用```类型之外的任何类型），```this```的值一直是```null```。但是由于把```this```的值设置为```null```没有任何意义，所以它被隐含的转换成全局对象。

让我们看个例子：

	function foo() {
	  return this;
	}

	foo(); // global

我们看到调用圆括号左侧是一个```引用```类型值（因为foo是一个标识符）：

	var fooReference = {
	  base: global,
	  propertyName: 'foo'
	};

相应地，```this```的值会被设置为```引用```类型值的base对象，也就是全局对象。

对于属性访问器也是类似：

	var foo = {
	  bar: function () {
	    return this;
	  }
	};

	foo.bar(); // foo

我们又得到了```引用```类型的值，其base的值是```foo```对象并且作为函数```bar```的```this```的值：

	var fooBarReference = {
	  base: foo,
	  propertyName: 'bar'
	};

但是，用另一种形式的调用表达式来调用同样的函数，那么```this```将会是其他值：

	var test = foo.bar;
	test(); // global

因为```test```，作为标识符，创建了其他```引用```类型的值，它的base属性(全局对象)被作为this的值：

	var testReference = {
	  base: global,
	  propertyName: 'test'
	};

注意，在ES5的[严格模式](http://dmitrysoshnikov.com/ecmascript/es5-chapter-2-strict-mode/)中```this```的值不会被[强制为](http://dmitrysoshnikov.com/ecmascript/es5-chapter-2-strict-mode/#codethiscode-value-restrictions)全局对象，而是被设置为```undefined```。

现在我们可以准确的说，为什么相同的函数通过不同形式的调用表达式进行调用，```this```值会不同－答案是因为它们位于不同的引用类型中间值：

	function foo() {
	  alert(this);
	}

	foo(); // global, because

	var fooReference = {
	  base: global,
	  propertyName: 'foo'
	};

	alert(foo === foo.prototype.constructor); // true

	// another form of the call expression

	foo.prototype.constructor(); // foo.prototype, because

	var fooPrototypeConstructorReference = {
	  base: foo.prototype,
	  propertyName: 'constructor'
	};

通过调用表达式的形式来动态确定```this```值的另一个（经典）例子：

	function foo() {
	  alert(this.bar);
	}

	var x = {bar: 10};
	var y = {bar: 20};

	x.test = foo;
	y.test = foo;

	x.test(); // 10
	y.test(); // 20


<span id="function-call-and-non-reference-type"></span>
###  函数调用和非引用类型

那么，就像我们提到过的，当调用括号左侧的值不是```引用```类型而是其他任意类型的话，```this```的值会自动地设置为```null```并且会成为全局对象。

让我们考虑下如下表达式：

	(function () {
	  alert(this); // null => global
	})();

在这个情况下，我们得到的是函数对象而不是```引用```类型对象（它不是标识符也不是属性访问器），相应地```this```值最终会被设置为全局对象。

更复杂的例子:

	var foo = {
	  bar: function () {
	    alert(this);
	  }
	};

	foo.bar(); // Reference, OK => foo
	(foo.bar)(); // Reference, OK => foo

	(foo.bar = foo.bar)(); // global?
	(false || foo.bar)(); // global?
	(foo.bar, foo.bar)(); // global?

那么，为什么中间结果应该为```引用```类型值的属性选择器，在某些调用中我们得到的```this```的值不是base对象（也就是```foo```）而是全局对象？

问题在于最后三个调用，在执行了某些操作之后，调用括号左侧的值已经不是引用类型了。

第一个例子显而易见－很明显那是```引用```类型，最后的结果```this```的值为base对象，也就是```foo```。

在第二个例子中存在一个分组操作符，这个操作符没有调用从```引用```类型值获取对象真实值的方法，也就是```GetValue```(请看[11.1.6](http://bclary.com/2004/11/07/#a-11.1.6)中的注解)。相应地，在分组操作符执行返回之后－我们仍然得到的是```引用```类型的值，这就是为什么```this```值又被设置成了base对象，也就是```foo```。

在第三个例子中，赋值操作符，不像分组操作符，调用了```GetValue```方法（请看[11.13.1](http://bclary.com/2004/11/07/#a-11.13.1)中的第三步）。函数返回后的值已经是函数对象了（并不是```引用```类型的值），这意味着```this```的值会被设置为```null```，也就是全局对象。

第四个和第五个例子也是类似－逗号操作符和逻辑或表达式调用了```GetValue```方法，相应地我们失去了引用类型的值而得到了函数类型的值；并且```this```的值又被设置成了全局对象。

<span id="-reference-type-and-null-this-value"></span>
###  引用类型和this的null值

有一种情况是当调用表达式确定调用括号的左侧是引用类型的值，但是```this```的值却被设置为了null，也就是全局对象。这个情况发生在当```引用```类型值的base对象是[活动对象(activation object)](http://dmitrysoshnikov.com/ecmascript/chapter-2-variable-object/#variable-object-in-function-context)的时候。

我们可以用在父函数中调用内部函数的例子来看看这个情况。像我们从[第二章](/chapter-2-variable-object.html)所了解的一样，局部变量，内部函数以及形参都是存储于对应函数的活动对象中：

	function foo() {
	  function bar() {
	    alert(this); // global
	  }
	  bar(); // the same as AO.bar()
	}

活动对象一直返回```this```的值－```null```（也就是说，用伪代码来说，```AO.bar()```等于```null.bar()```）。我们回过头再看下上面描述过的例子，这次```this```的值又被设置为了全局对象。

如果with对象包含一个函数名属性，那么在```with```语句块中进行函数调用的时候会出现例外的情况。with语句会把它的对象添加到[作用域链](http://dmitrysoshnikov.com/ecmascript/chapter-4-scope-chain/#affecting-on-scope-chain-during-code-execution)前面，也就是位于活动对象之前。相应地，获取```引用```类型的值（通过标识符或者属性访问器），我们得到的base对象不是活动对象而是```with```语句的对象。顺便说一下，它不只是与内部函数有关，也与全局函数有关，因为```with```对象是作用域链中优先级较高的对象（比全局对象或者活动对象）：

```
var x = 10;

with ({

  foo: function () {
    alert(this.x);
  },
  x: 20

}) {

  foo(); // 20

}

// because

var  fooReference = {
  base: __withObject,
  propertyName: 'foo'
};
```

相似的情况是调用扮演`catch`从句实参的函数：在这种情况下`catch`对象也被添加到作用域链前面，也就是位于活动对象或者全局对象之前。但是，这个给定的行为被认为是ECMA-262-3的一个bug，并且在新的标准－ECMA-262-5中被修复了，换句话说，这个活动中的`this`值应该是全局对象，而不是`catch`对象：

```
	try {
	  throw function () {
	    alert(this);
	  };
	} catch (e) {
	  e(); // __catchObject - in ES3, global - fixed in ES5
	}

	// on idea

	var eReference = {
	  base: __catchObject,
	  propertyName: 'e'
	};

	// but, as this is a bug
	// then this value is forced to global
	// null => global

	var eReference = {
	  base: global,
	  propertyName: 'e'
	};
```

相同的情况是递归调用[命名函数表达式](http://dmitrysoshnikov.com/ecmascript/chapter-5-functions/#feature-of-named-function-expression-nfe)（关于函数更多的细节请看[第五章、函数](http://dmitrysoshnikov.com/ecmascript/chapter-5-functions/)）。在函数第一次调用时候，base对象是父活动对象（或者全局对象），在递归调用的时候－base对象应该是特殊的对象，其中存储着函数表达式可有可无的名字。在这种情况下```this```的值也一直被设置为全局对象：

	(function foo(bar) {

	  alert(this);

	  !bar && foo(1); // "should" be special object, but always (correct) global

	})(); // global

<span id="this-value-in-function-called-as-the-constructor"></span>
###  构造函数中this的值

函数上下文中```this```值的变化还有一种情况－就是以构造函数的方式来调用函数：

	function A() {
	  alert(this); // newly created object, below - "a" object
	  this.x = 10;
	}

	var a = new A();
	alert(a.x); // 10

在这种情况下，[new](http://bclary.com/2004/11/07/#a-11.2.2)操作符会调用函数```A```内部的[[[Construct]]](http://bclary.com/2004/11/07/#a-13.2.2)方法，接下来在对象创建完成之后，调用函数```A```内部的[[[Call]]](http://bclary.com/2004/11/07/#a-13.2.1)方法，并把新创建的对象作为```this```的值。

<span id="manual-setting-of-this-value-for-a-function-call"></span>
###  调用函数时手动设置this值

在```Function.prototype```中定义了两个方法（因此它们可以被所有的函数访问到），通过它们可以手动制定函数调用时```this```的值。它们是```apply```和```call```方法。

这两个方法的第一个参数都是调用上下文中```this```的值。两者的区别不是很明显：对于```apply```来说第二个参数必须是数组（或者，类数组的对象，比如```argument```），相应地，```call```方法第二个参数可以接收任何参数；二者唯一相同的参数是第一个－```this```的值。

例子:

	var b = 10;

	function a(c) {
	  alert(this.b);
	  alert(c);
	}

	a(20); // this === global, this.b == 10, c == 20

	a.call({b: 20}, 30); // this === {b: 20}, this.b == 20, c == 30
	a.apply({b: 30}, [40]) // this === {b: 30}, this.b == 30, c == 40

<span id="conclusion"></span>
## 总结

在这篇文章里我们讨论了ECMAScript中```this```关键字的一些特性（与C++或Java相比来说，它们的确是特性）。我希望这篇文章可以对理解ECMAScript中```this```关键字如何工作有所帮助。还是和平时一样，欢迎留言，我很乐意回答你的问题。

<span id="-additional-literature"></span>
## 额外资料

* 10.1.7 – [This](http://bclary.com/2004/11/07/#a-10.1.7);
* 11.1.1 – [The this keyword](http://bclary.com/2004/11/07/#a-11.1.1);
* 11.2.2 – [The new operator](http://bclary.com/2004/11/07/#a-11.2.2);
* 11.2.3 – [Function calls](http://bclary.com/2004/11/07/#a-11.2.3).
