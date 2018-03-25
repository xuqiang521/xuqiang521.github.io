---
layout: post
title: "从指向看JavaScript"
date: 2018-03-17 
description: "this?原型/原型链？继承？闭包？我们不妨从指向这个概念去理解JavaScript中的这些点！"
tag: 合格前端
--- 

## 前言

开写前大家先来理解一下指向：指向，即目标方向、所对的方位。

很多人刚刚接触前端甚至一些“老”前端都经常会在JavaScript中所谓的难点，如this，原型，继承，闭包等这些概念中迷失了自我。接下来这篇文章会把我自己对于JavaScript中这些点通过指向的概念做个总结并分享给大家，希望可以帮助大家更好的了解这些所谓的难点。

## 一、this

this是什么？其实它本身就是一种指向。this指向可以分为以下几种情况

- 普通调用，this指向为调用者
- call/apply调用，this指向为当前thisArg参数
- 箭头函数，this指向为当前函数的this指向

这个怎么理解呢？接下来我会一一做解析。

### 1、普通调用

通俗理解一下，就是谁调用，则this便指向谁。这里又大致分为几种情况，分别为

#### 1.1、对象方法的调用
即某方法为某对象上的一个属性的属性，正常情况当改方法被调用的时候，this的指向则是挂载该方法的对象。废话不多说，直接看代码可能会更好的理解

```javascript
var obj = {
  a: 'this is obj',
  test: function () {
    console.log(this.a);
  }
}
obj.test();
// this is obj
```

#### 1.2、“单纯”函数调用
即该函数为自己独立的函数，而不是挂载到对象上的属性（window除外），也不会被当成构造函数来使用，而仅仅是当成函数来使用，此时的this指向则是window对象。例子如下

```javascript
var a = 'this is window'
function test () {
  console.log(this.a);
}
test();
// this is window
```
这个我们来理解一下，其实也很简单，我们都知道，window对象是全局对象。其实整个代码块等同于

```javascript
window.a = 'this is window'
window.test = function test () {
  console.log(this.a);
  // 此时是window为调用者，即this会指向window
}
window.test();
```
#### 1.3、构造函数调用

即该函数被当成构造函数来调用，此时的this指向该构造器函数的实例对象。我们来看一个例子，先上一个属于第二种情况的例子

```javascript
function test () {
  this.a = 'this is test';
  console.log(this.a);
  console.log(this);
}
test();
// this is test
// Window {}
```

按照上面的来理解，此时的this的确指向window对象，但是如果我换种形式，将其换成构造函数来调用呢，结果又会如何呢，直接上代码

```javascript
function Test () {
  this.a = 'this is test';
  console.log(this.a);
  console.log(this);
}
var test = new Test();
// this is test
// Test {a: 'this is test'}
```
OK，好像的确没有问题了，此时的this的确指向了该构造函数的实例对象。具体这里的一些解释后面我会在原型链继承里面详细讲解。

### 2、call/apply调用
#### 2.1、call调用
call方法形式，fun.call(thisArg[, arg1[, arg2[, ...]]])

- thisArg，当前this指向
- arg1[, arg2[, ...]]，指定的参数列表
详细介绍请猛戳 [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call)

示例代码如下

```javascript
function Test () {
  this.a = 'this is test';
  console.log(this.a);
  console.log(this);
}
function Test2 () {
  Test.call(this);
}
var test = new Test2();
// this is test
// Test2 {a: 'this is test'}
```

#### 2.2、apply调用
和call类似，唯一的一个明显区别就是call参数为多个，apply参数则为两个，第二个参数为数组或类数组形式， fun.apply(thisArg, [argsArray])

- thisArg，当前this指向
- 一个数组或者类数组对象，其中的数组元素将作为单独的参数传给fun函数
详细介绍请猛戳 [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply)

但是终究apply里面的数组参数会转变为call方法的参数形式，然后去走下面的步骤，这也是为什么call执行速度比apply快。这边详情有篇文章有介绍，[点击链接](https://zhuanlan.zhihu.com/p/27659836)。

另外，提及到call/apply，怎么能不提及一下bind呢，bind里面的this指向，会永远指向 bind 到的当前的 thisArg，即 context 上下文环境参数不可重写。这也是为什么 **a.bind(b).call(c)**，最终的 this 指向会是 b 的原因。至于为什么，其实就是 bind 实现实际上是通过闭包，并且配合 call/apply 进行实现的。具体的请参考bind MDN里面的用法及 Polyfill 实现。

### 3、箭头函数

首先需要介绍的一点就是，在箭头函数本身，它是没有绑定本身的this的，它的this指向为当前函数的this指向。怎么理解呢，直接上个代码看下

```javascript
function test () {
  (() => {
    console.log(this);
  })()
}
test.call({a: 'this is thisArg'})
// Object {a: 'this is thisArg'}
```

这样看联想上面的call/apply调用的理解，好像是没有问题了，那如果我设置一个定时器呢，会不是this指向会变成Window全局对象呢？答案肯定是不会的，因为箭头函数里面的this特殊性，它依旧会指向当前函数的this指向。不多BB，直接看代码

```javascript
function test () {
  setTimeout(() => {
    console.log(this);
  }, 0)
}
test.call({a: 'this is obj'})
// Object {a: 'this is obj'}
```

当然普通函数使用setTimeout的话会让this指向指向Window对象的。demo代码如下

```javascript
function test () {
  setTimeout(function () {
    console.log(this);
  }, 0)
}
test.call({a: 'this is obj'})
// Window {...}
```

这里可能会牵扯到setTimeout的一些点了，具体这里我就不讲了，想深入了解的[猛戳这里](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/setTimeout)

箭头函数里面还有一些特殊的点，这里由于只提及this这一个点，其他比如不绑定arguments，super(ES6)，抑或 new.target(ES6)，他们都和this一样，他会找寻到当前函数的arguments等。

关于箭头函数里面的this这里也有详细的介绍，想深入了解的可以自行阅读

- [英文原版（需翻墙）](http://blog.getify.com/arrow-this)
- [中文翻译版](http://www.cnblogs.com/vajoy/p/4902935.html)

## 二、原型/原型链
其实我们一看到原型/原型链都能和继承联想到一起，我们这里就把两块先拆开来讲解，这里我们就先单独把原型/原型链拎出来。首先我们自己问一下自己，什么是原型？什么是原型链？

- 原型：即每个function函数都有的一个prototype属性。
- 原型链：每个对象和原型都有原型，对象的原型指向原型对象，而父的原型又指向父的父，这种原型层层连接起来的就构成了原型链。
好像说的有点绕，其实一张图可以解释一切

![](/images/blog/0317/proto.png)

那么这个东西有怎么和指向这个概念去联系上呢？其实这里需要提及到的一个点，也是上面截图中存在的一个点，就是 **__proto__**，我喜欢把其称为原型指针。终归到头，**prototype** 只不过是一个属性而已，它没有什么实际的意义，最后能做原型链继承的还是通过 **__proto__** 这个原型指针来完成的。我们看到的所谓的继承只不过是将需要继承的属性挂载到继承者的 **prototype** 属性上面去的，实际在找寻继承的属性的时候，会通过 **__proto__** 原型指针一层一层往上找，即会去找**__proto__** 原型指针它的一个指向。看个 demo

```javascript
function Test () {
  this.a = 'this is Test';
}
Test.prototype = {
  b: function () {
    console.log("this is Test's prototype");
  }
}
function Test2 () {
  this.a = 'this is Test2'
}
Test2.prototype = new Test();
var test = new Test2();
test.b();
console.log(test.prototype);
console.log(test);
```

其执行结果如下

![](/images/blog/0317/proto2.png)

更多关于继承的点，这里就不提及了，我会在继承这一章节做详细的讲解。那么“单独”关于原型/原型链的点就这些了。
总结：原型即 **prototype**，它只是所有 **function** 上的一个属性而已，真正的“大佬”是 **__proto__**，“大佬”指向谁，谁才能有言语权（当然可能因为“大佬”过于霸道，所以在 **ECMA-262** 之后才被 **Standard** 化）。

![](/images/blog/0317/funny.png)

## 三、继承

关于继承，之前我有写过一篇博文对继承的一些主流方式进行过总结。想详细了解的请点击传送门。这里我们通过指向这个概念来重新理解一下继承。这里咱就谈两个万变不离其宗的继承方式，一个是构造函数继承，一个是原型链继承。

### 1、构造函数继承
其实就是上面提及到的通过 call/apply 调用，将 this 指向变成 thisArg，具体看上面的解释，这里直接上代码

```javascript
function Test () {
  this.a = 'this is test';
  console.log(this.a);
  console.log(this);
}
function Test2 () {
  Test.apply(this)
  // or Test.apply(this)
}
var test = new Test2();
// this is test
// Test2 {a: 'this is test'}
```

### 2、原型链继承

一般情况，我们做原型链继承，会通过子类 **prototype** 属性等于（指向）父类的实例。即

```javascript
Child.prototype = new Parent();
```

那么这样的做法具体是怎么实现原型链继承的呢？

首先在讲解继承前，我们需要 get 到一个点，那就是**对象 { }** 它内部拥有的一些属性，这里直接看张图

![](/images/blog/0317/obj.png)

如上图所示，我们看到**对象 { }** 它本身拥有的属性就是上面我们提及到的 **__proto__** 原型指针以及一些方法。
接下来我先说一下 **new** 关键字具体做的一件事情。其过程大致分为三步，如下

```javascript
var obj= {}; // 初始化一个对象obj
obj.__proto__ = Parent.prototype; // 将obj的__proto__原型指针指向父类Parent的prototype属性
Parent.call(obj); // 初始化Parent构造函数
```
从这里我们看出来，相信大家也能理解为什么我在上面说 **__proto__** 才是真正的“大佬”。

这里我额外提一件我们经常干的“高端”的事情，那就是通过原型 **prototype** 做 **monkey patch**。即我想在继承父类方法的同时，完成自己独立的一些操作。具体代码如下

```javascript
function Parent () {
  this.a = 'this is Parent'
}
Parent.prototype = {
  b: function () {
    console.log(this.a);
  }
}
function Child () {
  this.a = 'this is Child'
}
Child.prototype = {
  b: function () {
    console.log('monkey patch');
    Parent.prototype.b.call(this);
  }
}
var test = new Child()
test.b()
// monkey patch
// this is Child
```

这个是我们对于自定义的类进行继承并重写，那么如果是类似 **Array**，**Number**，**String** 等内置类进行继承重写的话，结果会是如何呢？关于这个话题我也有写过一篇博文进行过讲解，[传送门](https://my.oschina.net/qiangdada/blog/911252)

## 四、闭包

对于闭包，我曾经也做过总结和分享，简单的一些东西和概念这里不提及了，想了解的可以[猛戳这里](https://my.oschina.net/qiangdada/blog/753961)。和原型链那节一样，这里会摒弃掉原来的一些看法，这里我依旧通过代入 **指向** 这个概念来进行理解。

一般情况下，我们理解闭包是这样的：**“为了可以访问函数内的局部变量而定义的内部函数”**。

JavaScript 语言特性，每一个 **function** 内都有一个属于自己的执行上下文，即特定的 context 指向。

![](/images/blog/0317/context.png)

内层的 context 上下文总能访问到外层 context 上下文中的变量，即每次内部的作用域可以往上层查找直到访问到当前所需访问的变量。例子如下

```javascript
var a = 'this is window'
function test () {
  var b = 'this is test'
  function test2 () {
    var c = 'this is test2';
    console.log(a);
    console.log(b);
    console.log(c);
  }
  test2();
}
test();
// this is window
// this is test
// this is test2
```

但是如果反过来访问的话，则不能进行访问，即 **变量访问的指向是当前context上下文的指向的相反方向，且不可逆。**如下

```javascript
function test () {
  var b = 'this is test';
}
console.log(b); // Uncaught ReferenceError: b is not defined
```

这里用一个非常常见的情况作为例子，即 **for循环** 配合 **setTimeout** 的异步任务，如下

```javascript
function test () {
  for (var i = 0; i < 4; i++) {
    setTimeout(function () {
      console.log(i);
    }, 0)
  }
}
test();
```

看到上面的例子，我们都知道说：“答案会打印4次4”。那么为什么会这样呢？我想依次打印0，1，2，3又该怎么做呢？

相信很多小伙伴们都会说，用闭包呀，就能实现了呀。对没错，的确用闭包就能实现。那么为什么出现这种情况呢？

这里我简单提一下，首先这边牵扯到两个点，一个就是 for 循环的同步任务，一个就是 **setTimeout** 的异步任务，在JavaScript线程中，因为本身 JavaScript 是单线程，这个特点决定了其正常的脚本执行顺序是按照文档流的形式来进行的，即从上往下，从左往右的这样方向。每次脚本正常执行时，但凡遇到异步任务的时候，都会将其 set 到一个 **task queue（任务队列）**中去。然后在执行完同步任务之后，再来执行队列任务中的异步任务。 

当然对于不同的异步任务，执行顺序也会不一样，具体就看其到底属于哪个维度的异步任务了。这里我就不详细扯 **Event Loop** 了，想更详细的了解请[戳这里](https://mp.weixin.qq.com/s?__biz=MjM5MTA1MjAxMQ==&mid=2651226694&idx=1&sn=01908e1c5089010733e723c99947b311&chksm=bd495bc28a3ed2d4d92c024910eb2b0367d0b22ee8e2587fee9253a359ebf99dba63338f3ccb&mpshare=1&scene=1&srcid=0715cSwmYEIhrnBgHSKPcp4V&key=31a3a388ed6fa8730fef6b2af30c13a7f5980b0c24be77b2ed2c155b6640f59771708ea75667ab90fe26d9f8d82e5a7841de72bf1c749dcdeec997f40e18f77bfeb9f82bdd4216549eb2fe9894acfb2e&ascene=0&uin=MTc2Mjk2NzU%3D&devicetype=iMac+MacBookPro12%2C1+OSX+OSX+10.11.6+build(15G31)&version=12020010&nettype=WIFI&fontScale=100&pass_ticket=21mmnEEP0ZrrNHJG%2BPgYOJW1ALluEzetU8oy6dySXLI%3D)

回到上面我们想要实现的效果这个问题上来，我们一般处理方法是利用闭包进行参数传值，代码如下

```javascript
function test () {
  for (var i = 0; i < 4; i++) {
    (function (e) {
      setTimeout(function () {
        console.log(e);
      }, 0)
    })(i)
  }
}
test();
// 0 -> 1 -> 2 -> 3
```

**循环当中，匿名函数会立即执行，并且会将循环当前的 i 作为参数传入，将其作为当前匿名函数中的形参e的指向，即会保存对 i 的引用，它是不会被循环改变的。**

当然还有一种常见的方式可以实现上面的效果，即从自执行匿名函数中返回一个函数。代码如下

```javascript
function test () {
  for(var i = 0; i < 4; i++) {
    setTimeout((function(e) {
      return function() {
        console.log(e);
      }
    })(i), 0)
  }
}
test();
```

更多高阶闭包的写法这里就不一一介绍了，想了解的小伙伴请自行搜索。

文章到此差不多就要结束了

![](/images/blog/0317/funny2.png)

可是我也没办法，的确要结束了。

## 总结

首先基本上 JavaScript 中所涉及的所谓的难点，在本文中都通过 **指向** 这个概念进行了通篇的解读，当然这是我个人对于 JavaScript 的一些理解，思路仅供参考。如果有什么不对的地方，欢迎各位小伙伴指出。

其实写该博文的好多次，我想把所有的知识点全部串起来进行讲解，但又怕效果不好，所以做了一一的拆解，也进行了混合的运用。具体能领悟到多少，就要看小伙伴你们自己的了。
