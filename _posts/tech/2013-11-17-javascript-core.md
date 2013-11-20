---
layout: post
title: JavaScript核心
city: 南京
tags: [tech]
---

1. 对象An object
2. A prototype chain
3. Constructor
4. Execution context stack
5. Execution context
6. Variable object
7. Activation object
8. Scope chain
9. Closures
10. This value
11. Conclusion


这篇文章是[「深入ECMA-262-3」](http://dmitrysoshnikov.com/tag/ecma-262-3/)系列的一个概览和摘要。每个部分都包含了对应章节的链接，所以你可以阅读它们以获取更深的理解。

面向读者：经验丰富的程序员，专家。

我们以思考对象的概念做为开始，这是ECMAScript的基础。

#对象

ECMAScript做为一个高度抽象的面向对象语言，是处理对象的。里边也有基本类型，但是，当需要的时候，它们也会被转换成对象。

	一个对象就是一个属性集合，并拥有一个独立的prototype（原型）对象。这个prototype可以是一个对象或者null。

让我们看一个关于对象的基本例子。一个对象的prototype是以内部的[[Prototype]]属性来引用的。但是，在示意图里边我们将会使用```__<internal-property>__ ```下划线标记来替代两个括号，对于prototype对象来说是：```__proto__```。

对于以下代码：

	var foo = {
	  x: 10,
	  y: 20
	};

我们拥有一个这样的结构，两个明显的自身属性和一个隐含的```__proto__```属性，这个属性是对```foo```原型对象的引用：

[![](http://{{ site.cdn }}/images/tech/basic-object.png)](http://{{ site.cdn }}/images/tech/basic-object.png)

这些prototype有什么用？让我们以原型链（prototype chain）的概念来回答这个问题。

#原型链

原型对象也是简单的对象并且可以拥有它们自己的原型。如果一个原型对象的原型是一个非null的引用，那么以此类推，这就叫作原型链。

	原型链是一个用来实现继承和共享属性的有限对象链。

考虑这么一个情况，我们拥有两个对象，它们之间只有一小部分不同，其他部分都相同。显然，如果是一个设计良好的系统，我们将会重用相似的功能/代码而不是在每个单独的对象中重复它。在基于类的系统中，这个代码重用风格叫作_类继承_　－　你把相似的功能放入class ```A```中，然后class ```B```和class ```C```继承class ```A```，并且自身拥有一些小的额外的改动。

ECMAScript中没有类的概念。但是，代码重用的风格并没有太多不同（尽管从某些方面来说那比基于类（class-based）的方式要更加灵活）并且其通过原型链来实现。这种继承方式叫作委托继承（或者，靠近ECMAScript一些，叫作原型继承）。

跟例子中的类```A```，```B```，```C```相似，在ECMAScript中你创建对象：```a```，```b```，```c```。于是，对象```a```中存储对象```b```和```c```中通用的部分。然后```b```和```c```只存储它们自身的额外属性或者方法。

	var a = {
	  x: 10,
	  calculate: function (z) {
	    return this.x + this.y + z
	  }
	};
 
	var b = {
	  y: 20,
	  __proto__: a
	};
 
	var c = {
	  y: 30,
	  __proto__: a
	};
 
	// call the inherited method
	b.calculate(30); // 60
	c.calculate(40); // 80
	
足够简单，是不是？我们看到```b```和```c```访问到了在对象```a```中定义的```calculate```方法。这是通过原型链实现的。

规则很简单：如果一个属性或者一个方法在对象自身中无法找到（比如，对象自身没有一个那样的属性），然后它会尝试在原型链中寻找这个属性/方法。如果这个属性在原型中没有查找到，那么将会查找这个原型的原型，以此类推，遍历整个原型链（当然这在类继承中也是一样的，当解析一个继承的方法的时候－我们遍历__class链__（ class chain））。第一个查找到的名字相同的属性/方法会被使用。因此，一个查找到的属性叫作继承属性。如果在遍历了整个原型链之后还是没有查找到这个属性的话，返回```undefined```值。

注意，继承方法中所使用的```this```的值被设置为原始对象，而并不是在其中查找到这个方法的（原型）对象。比如，在上面的例子中```this.y```取的是```b```和```c```中的值，而不是```a```中的。但是，```this.x```是取的是```a```中的值，并且又一次通过原型链机制完成。

如果没有明确为一个对象指定原型，那么它将会使用```__proto__```的默认值－```Object.prototype```。Object.prototype对象自身也有一个```__proto__```属性，这是原型链的终点并且值为```null```。

下一张图展示了对象```a```，```b```，```c```之间的继承层级：

[![](http://{{ site.cdn }}/images/tech/prototype-chain.png)](http://{{ site.cdn }}/images/tech/prototype-chain.png)


注意：
ES5标准化了一个实现原型继承的可选方法，即使用Object.create函数：

	var b = Object.create(a, {y: {value: 20}});
	var c = Object.create(a, {y: {value: 30}});

你可以在[对应的章节](http://dmitrysoshnikov.com/ecmascript/es5-chapter-1-properties-and-property-descriptors/#new-api-methods)获取到更多关于ES5新API的信息。
ES6标准化了 ```__proto__```属性，并且它可以在对象初始化的时候使用。

通常情况下需要对象拥有相同或者相似的状态结构（比如，相同的属性集合），赋以不同的状态值。在这个情况下我们可能需要使用构造函数，其以指定的模式来创造对象。

#构造函数

除了以指定模式创建对象之外，构造函数也做了另一个有用的事情－它自动地为新创建的对象设置一个原型对象。这个原型对象存储在```ConstructorFunction.prototype```属性中。

比如，我们可以使用构造函数来重写上一个拥有对象```b```和对象```c```的例子。因此，对象```a```（一个原型对象）的角色由```Foo.prototype```来扮演：

	// a constructor function
	function Foo(y) {
	  // which may create objects
	  // by specified pattern: they have after
	  // creation own "y" property
	  this.y = y;
	}
 
	// also "Foo.prototype" stores reference
	// to the prototype of newly created objects,
	// so we may use it to define shared/inherited
	// properties or methods, so the same as in
	// previous example we have:
 
	// inherited property "x"
	Foo.prototype.x = 10;
 
	// and inherited method "calculate"
	Foo.prototype.calculate = function (z) {
	  return this.x + this.y + z;
	};
 
	// now create our "b" and "c"
	// objects using "pattern" Foo
	var b = new Foo(20);
	var c = new Foo(30);
 
	// call the inherited method
	b.calculate(30); // 60
	c.calculate(40); // 80
 
	// let's show that we reference
	// properties we expect
 
	console.log(
 
	  b.__proto__ === Foo.prototype, // true
	  c.__proto__ === Foo.prototype, // true
 
	  // also "Foo.prototype" automatically creates
	  // a special property "constructor", which is a
	  // reference to the constructor function itself;
	  // instances "b" and "c" may found it via
	  // delegation and use to check their constructor
 
	  b.constructor === Foo, // true
	  c.constructor === Foo, // true
	  Foo.prototype.constructor === Foo // true
 
	  b.calculate === b.__proto__.calculate, // true
	  b.__proto__.calculate === Foo.prototype.calculate // true
 
	);

这个代码可以表示为如下关系：

[![](http://{{ site.cdn }}/images/tech/constructor-proto-chain.png)](http://{{ site.cdn }}/images/tech/constructor-proto-chain.png)

这张图又一次说明了每个对象都有一个原型。构造函数```Foo```也有自己的```__proto__```，值为```Function.prototype```，```Function.prototype```也通过其```__proto__```属性关联到```Object.prototype```。因此，重申一下，```Foo.prototype```就是```Foo```的一个明确的属性，其与对象b和对象c的原型相关。

正式点来说，如果思考一下分类的概念（并且我们已经对```Foo```进行了分类），那么构造函数和原型对象的整合可以叫作「类」。实际上，举个例子，Python的第一级（first-class）动态类（dynamic classes）显然是以同样的```属性/方法```处理方案来实现的。从这个角度来说，Python中的类就是ECMAScript使用的委托继承的一个语法糖。

注意: 在ES6中「类」的概念被标准化了，并且实际上以一种构建在构造函数上面的语法糖来实现，就像上面描述的一样。从这个角度来看原型链成为了类继承的一种实现方式：

	// ES6
	class Foo {
	  constructor(name) {
	    this._name = name;
	  }
 
	  getName() {
	    return this._name;
	  }
	}
 
	class Bar extends Foo {
	  getName() {
	    return super.getName() + ' Doe';
	  }
	}
 
	var bar = new Bar('John');
	console.log(bar.getName()); // John Doe
	
有关这个主题的完整、详细的解释可以在ES3系列的第七章找到。分为两个部分：[7.1章 面向对象. 基本理论](http://dmitrysoshnikov.com/ecmascript/chapter-7-1-oop-general-theory/)，在那里你将会找到对各种面向对象的范例、风格以及它们和ECMAScript对比的描述，然后在[7.2章　面向对象. ECMAScript实现](http://dmitrysoshnikov.com/ecmascript/chapter-7-2-oop-ecmascript-implementation/)，是对ECMAScript中面向对象的介绍。

现在，我们知道了对象的基本方面之后，让我们看看运行时程序执行（runtime program execution）在ECMAScript中是如何实现的。这叫作执行上下文堆栈（execution context stack），其中的每个元素也可以抽象成为一个对象。是的，ECMAScript几乎在任何地方都和对象的概念打交道;)

#执行上下文堆栈

这里有三种类型的ECMAScript代码：全局代码、函数代码和eval代码。每个代码是在其执行上下文（execution context）中被求值的。这里只有一个全局上下文，可能有多个函数执行上下文以及eval执行上下文。对一个函数的每次调用，会进入到函数执行上下文中，并对函数代码类型进行求值。每次对eval函数进行调用，会进入```eval```执行上下文并对其代码进行求值。

注意，一个函数可能会创建无数的上下文，因为对函数的每次调用（即使这个函数递归的调用自己）都会生成一个具有新状态的上下文：

	function foo(bar) {}
 
	// call the same function,
	// generate three different
	// contexts in each call, with
	// different context state (e.g. value
	// of the "bar" argument)
 
	foo(10);
	foo(20);
	foo(30);

一个执行上下文可能会触发另一个上下文，比如，一个函数调用另一个函数（或者在全局上下文中调用一个全局函数），等等。从逻辑上来说，这是以堆栈的形式实现的，它叫作执行上下文堆栈。

一个触发其他上下文的上下文叫作caller。被触发的上下文叫作callee。callee在同一时间可能是一些其他callee的caller（比如，一个在全局上下文中被调用的函数，之后调用了一些内部函数）。

当一个caller触发（调用）了一个callee，这个caller会暂缓它的执行，然后把控制权传递给callee。这个callee被push到栈中，并成为一个运行中（活动的）执行上下文。在callee的上下文结束后，它会把控制权返回给caller，然后caller的上下文继续执行（它可能触发其他上下文）直到它结束，以此类推。callee可能简单的返回或者由于异常而退出。一个抛出的但是没有被捕获的异常可能退出（从栈中pop）一个或者多个上下文。

比如，所有ECMAScript程序的运行时可以用执行上下文（EC）堆栈来表示，栈顶是当前活跃(active)上下文：

[![](http://{{ site.cdn }}/images/tech/ec-stack.png)](http://{{ site.cdn }}/images/tech/ec-stack.png)

当程序开始的时候它会进入全局执行上下文，此上下文位于栈底并且是栈中的第一个元素。然后全局代码进行一些初始化，创建需要的对象和函数。在全局上下文的执行过程中，它的代码可能触发其他（已经创建完成的）函数，这些函数将会进入它们自己的执行上下文，向栈中push新的元素，以此类推。当初始化完成之后，运行时系统（runtime system）就会等待一些事件（比如，用户鼠标点击），这些事件将会触发一些函数，从而进入新的执行上下文中。

在下个图中，用一些函数上下文```EC1```和全局上下文```Global EC```，当```EC1```进入和退出全局上下文的时候下面的栈将会发生变化：

[![](http://{{ site.cdn }}/images/tech/ec-stack-changes.png)](http://{{ site.cdn }}/images/tech/ec-stack-changes.png)

这就是ECMAScript的运行时系统如何真正地管理代码的执行的。

更多有关ECMAScript中执行上下文的信息可以在对应的[章节1 执行上下文](http://dmitrysoshnikov.com/ecmascript/chapter-1-execution-contexts/)中获取。

像我们所说的，栈中的每个执行上下文都可以用一个对象来表示。让我们来看看它的结构以及一个上下文到底需要什么状态（什么属性）来执行它的代码。

#执行上下文

一个执行上下文可以抽象的表示为一个简单的对象。每一个执行上下文拥有一些属性用来跟踪和它相关的代码的执行过程。在下图中展示了一个上下文的结构：

[![](http://{{ site.cdn }}/images/tech/execution-context.png)](http://{{ site.cdn }}/images/tech/execution-context.png)

除了这三个必需的属性（一个变量对象（variable objec），一个```this```值以及一个作用域链（scope chain））之外，执行上下文可以拥有任何附加的状态，这取决于实现。

让我们详细看看上下文中的这些重要的属性。

#变量对象

	变量对象是与执行上下文相关的数据的作用范围。它是一个与上下文相关的特殊对象，其中存储了在上下文中定义的变量和函数声明。

注意，函数表达式（与函数声明比较）不包含在变量对象之中。

变量对象是一个抽象概念。对于不同的上下文类型，在物理上，是使用不同的对象。比如，在全局上下文中变量对象就是全局对象本身（这就是为什么我们可以通过全局对象的属性名来关联全局变量）。

让我们在全局执行上下文中考虑下面这个例子：

	var foo = 10;
 
	function bar() {} // function declaration, FD
	(function baz() {}); // function expression, FE
 
	console.log(
	  this.foo == foo, // true
	  window.bar == bar // true
	);
 
	console.log(baz); // ReferenceError, "baz" is not defined
	
之后，全局上下文的变量对象（variable objec，简称VO）将会拥有如下属性：

[![](http://{{ site.cdn }}/images/tech/variable-object.png)](http://{{ site.cdn }}/images/tech/variable-object.png)

再看一遍，函数```baz```是一个函数表达式，没有被包含在变量对象之中。这就是为什么当我们想要在函数自身之外访问它的时候会出现```ReferenceError```。

注意，与其他语言（比如C/C++）相比，在ECMAScript中只有函数可以创建一个新的作用域。在函数作用域中定义的变量和内部函数在外边不是直接可见的，而且并不污染全局变量对象。

使用```eval```我们也会进入一个新的（eval类型）执行上下文。无论如何，```eval```使用全局的变量对象或者使用caller（比如```eval```被调用时所在的函数）的变量对象。

那么关于函数和它们的变量对象是怎么样的？在函数上下文中，变量对象是以活动对象（activation object）来表示的。

#活动对象

当一个函数被caller所触发（被调用），一个特殊的对象，叫作活动对象（activation object）将会被创建。这个对象中包含形参和那个特殊的```arguments```对象（是对形参的一个映射，但是值是通过索引来获取）。活动对象之后会做为函数上下文的变量对象来使用。

比如，一个函数的变量对象就是一个相同简单的变量对象，但是除了变量和函数声明之外，它也存储了形参和```arguments```对象并叫作活动对象。

考虑如下例子：

	function foo(x, y) {
	  var z = 30;
	  function bar() {} // FD
	  (function baz() {}); // FE
	}
 
	foo(10, 20);
	
我们看下函数```foo```的上下文中的活动对象（activation object，简称AO）：

[![](http://{{ site.cdn }}/images/tech/activation-object.png)](http://{{ site.cdn }}/images/tech/activation-object.png)

并且函数表达式baz还是没有被包含在变量/活动对象中。

对于这个主题的所有细节方面（像变量和函数声明的提升问题（hoisting））的完整描述可以在同名的章节[第二章　变量对象](http://dmitrysoshnikov.com/ecmascript/chapter-2-variable-object/)中找到。

注意，在ES5中变量对象和活动对象被并入了词法环境模型（lexical environments model），详细的描述可以在[对应的章节](http://dmitrysoshnikov.com/ecmascript/es5-chapter-3-2-lexical-environments-ecmascript-implementation/)找到。

然后我们向下一个部分前进。众所周知，在ECMAScript中我们可以使用内部函数，然后在这些内部函数我们可以引用外层函数的变量或者全局上下文中的变量。当我们把变量对象命名为上下文的作用域对象，与上面讨论的原型链相似，这里有一个叫作作用域链的东西。

#作用域链

	作用域链是一个对象列表，上下文代码中出现的标识符在这个列表中进行查找。

这个规则还是与原型链同样简单以及相似：如果一个变量在函数自身的作用域（在自身的变量/活动对象）中没有找到，那么将会查找它父函数（外层函数）的变量对象，以此类推。

就上下文而言，标识符指的是：变量名称，函数声明，形参，等等。当一个函数在其代码中引用一个不是局部变量（或者局部函数或者一个形参）的标识符，那么这个标识符就叫作自由变量。搜索这些自由变量正好就要用到作用域链。

在通常情况下，作用域链是一个包含所有父（函数）变量对象加上（在作用域链头部）函数自身变量/活动对象的一个列表。但是，这个作用域链也可以包含任何其他对象，比如，在上下文执行过程中动态加入到作用域链中的对象－像with对象或者特殊的catch从句（catch-clauses）对象。

当解析（查找）一个标识符，会从作用域链中的活动对象开始查找，然后（如果这个标识符在函数自身的活动对象中没有被查找到）向作用域链的上一层查找－重复这个过程，就和原型链一样。

	var x = 10;
 
	(function foo() {
	  var y = 20;
	  (function bar() {
	    var z = 30;
	    // "x" and "y" are "free variables"
	    // and are found in the next (after
	    // bar's activation object) object
	    // of the bar's scope chain
	    console.log(x + y + z);
	  })();
	})();

我们可以假设通过隐式的```__parent__```属性来和作用域链对象进行关联，这个属性指向作用域链中的下一个对象。这个方案可能在[真实的Rhino代码](http://dmitrysoshnikov.com/ecmascript/chapter-2-variable-object/#feature-of-implementations-property-__parent__)中经过了测试，并且这个技术很明确得被用于ES5的词法环境中（在那里被叫作```outer```连接）。作用域链的另一个表现方式可以是一个简单的数组。利用```__parent__```概念，我们可以用下面的图来表现上面的例子（并且父变量对象存储在函数的```[[Scope]]```属性中）：

[![](http://{{ site.cdn }}/images/tech/scope-chain.png)](http://{{ site.cdn }}/images/tech/scope-chain.png)

在代码执行过程中，作用域链可以通过使用```with```语句和```catch```从句对象来增强。并且由于这些对象是简单的对象，它们可以拥有原型（和原型链）。这个事实导致作用域链查找变为两个维度：（1）首先是作用域链连接，然后（2）在每个作用域链连接上－深入作用域链连接的原型链（如果此连接拥有原型）。

对于这个例子：

	Object.prototype.x = 10;
 
	var w = 20;
	var y = 30;
 
	// in SpiderMonkey global object
	// i.e. variable object of the global
	// context inherits from "Object.prototype",
	// so we may refer "not defined global
	// variable x", which is found in
	// the prototype chain
 
	console.log(x); // 10
 
	(function foo() {
 
	  // "foo" local variables
	  var w = 40;
	  var x = 100;
 
	  // "x" is found in the
	  // "Object.prototype", because
	  // {z: 50} inherits from it
 
	  with ({z: 50}) {
	    console.log(w, x, y , z); // 40, 10, 30, 50
	  }
 
	  // after "with" object is removed
	  // from the scope chain, "x" is
	  // again found in the AO of "foo" context;
	  // variable "w" is also local
	  console.log(x, w); // 100, 40
 
	  // and that's how we may refer
	  // shadowed global "w" variable in
	  // the browser host environment
	  console.log(window.w); // 20
 
	})();
	
我们可以给出如下的结构（确切的说，在我们查找```__parent__```连接之前，首先查找```__proto__```链）：

[![](http://{{ site.cdn }}/images/tech/scope-chain-with.png)](http://{{ site.cdn }}/images/tech/scope-chain-with.png)

注意，不是在所有的实现中全局对象都是继承自```Object.prototype```。上图中描述的行为（从全局上下文中引用「未定义」的变量```x```）可以在诸如SpiderMonkey引擎中进行测试。

由于所有父变量对象都存在，所以在内部函数中获取父函数中的数据没有什么特别－我们就是遍历作用域链去解析（搜寻）需要的变量。就像我们上边提及的，在一个上下文结束之后，它所有的状态和它自身都会被销毁。在同一时间父函数可能会返回一个内部函数。而且，这个返回的函数之后可能在另一个上下文中被调用。如果自由变量的上下文已经「消失」了，那么这样的调用将会发生什么？通常来说，有一个概念可以帮助我们解决这个问题，叫作（词法）闭包，其在ECMAScript中就是和作用域链的概念紧密相关的。

#闭包

在ECMAScript中，函数是第一级（first-class）对象。这个术语意味着函数可以做为参数传递给其他函数（在那种情况下，这些参数叫作"funargs"，是"functional arguments"的简称）。接收"funargs"的函数叫作高阶函数或者，靠近数学一些，叫作高阶操作符。同样函数也可以从其他函数中返回。返回其他函数的函数叫作以函数为值（function valued）的函数（或者叫作拥有函数类值的函数（functions with functional value））。


There are two conceptual problems related with “funargs” and “functional values”. And these two sub-problems are generalized in one which is called a “Funarg problem” (or “A problem of a functional argument”). And exactly to solve the complete “funarg problem”, the concept of closures was invented. Let’s describe in more detail these two sub-problems (we’ll see that both of them are solved in ECMAScript using a mentioned on figures [[Scope]] property of a function).

First subtype of the “funarg problem” is an “upward funarg problem”. It appears when a function is returned “up” (to the outside) from another function and uses already mentioned above free variables. To be able access variables of the parent context even after the parent context ends, the inner function at creation moment saves in it’s [[Scope]] property parent’s scope chain. Then when the function is activated, the scope chain of its context is formed as combination of the activation object and this [[Scope]] property (actually, what we’ve just seen above on figures):

	Scope chain = Activation object + [[Scope]]

Notice again the main thing — exactly at creation moment — a function saves parent’s scope chain, because exactly this saved scope chain will be used for variables lookup then in further calls of the function.

	function foo() {
	  var x = 10;
	  return function bar() {
	    console.log(x);
	  };
	}
	
	// "foo" returns also a function
	// and this returned function uses
	// free variable "x"
 
	var returnedFunction = foo();
 
	// global variable "x"
	var x = 20;
 
	// execution of the returned function

	returnedFunction(); // 10, but not 20

This style of scope is called the static (or lexical) scope. We see that the variable x is found in the saved [[Scope]] of returned bar function. In general theory, there is also a dynamic scope when the variable x in the example above would be resolved as 20, but not 10. However, dynamic scope is not used in ECMAScript.

