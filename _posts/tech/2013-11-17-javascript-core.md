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


这篇文章是[「深入ECMA-262-3」]（http://dmitrysoshnikov.com/tag/ecma-262-3/）系列的一个概览和摘要。每个部分都包含了对应章节的链接，所以你可以阅读它们以获取更深的理解。

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

