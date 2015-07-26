layout: title
title: "把书看薄-javascript模式"
date: 2015-03-15 16:06:06
tags: 读书笔记
---
#模式
我理解的模式是一种解决方案，用来更好的解决某类问题。也可以认为是最佳实践的一种体现。
#javascript模式
在软件行业，由于抽象的原因，模式多于面向对象挂钩，虽然javascript不是一门纯正面向对象的语言，但是它有原型链，有_一等公民_ 函数，有object对象，融在一起，完全可以实现面向对象。这也是这本书所探讨的话题。
_在没有说明的情况下，都是ES3的内容_
<!--more-->
##第一章 简介
简单地介绍，js基础好的人可以直接跳过
##第二章 基本技巧
+ 全局变量
  - 注意全局变量带来的弊端，这个话题就不用过多讨论了，全局变量在有了模式、命名空间之后是被抛弃的东西。
  - 下面这种形式要引起注意
`var a = b = 0;//b会被放在全局变量下，因为优先级的问题，这句话可以简单解释为：var a; window.b = 0; a = b; `
  - 通过var 定义的全局变量不可被delete掉，而未通过var定义的可以，当然，未通过var定义的在ES5严格模式会报错，*不被推荐*。

+ for循环
	- 普通for循环应该把数组长度缓存，以免每次判断的时候都需要取一次数组长度，这对于性能的提升很有帮助。
	- i--比i++更受推荐，因为同0比较比同数组程度更有效率。
+ for in循环
    - 需要注意的就是for in循环的时候会把原型链里面的属性带出来，如果for in的对象是一个实例，而且你不希望把它原型链中的数据带出来，就需要用 _hasOwnProperty_来过滤。
+ 推荐使用 _===_ 来增加代码的健壮性
+ eval是不推荐的，尤其是ajax回json用eval去执行不对格式校验会有安全隐患。实在是要使用eval也推荐用new Function();因为他有自己的作用域。
+ parseInt后面必须带上第二个参数，因为解析_"07"_之类的不指定进制会导致解析成8进制。

##第三章 字面量和构造函数
+ 不使用new Object()和new Array()来创建对象/数组，比如new Array带来的问题：
<pre>
new Array[3]//[undefined,undefined,undefined];
new Array[3.14]//error

</pre>     
+ 检测数组：
    - Array.isArray();//ECAMAScript 5;
    - Object.prototype.toString.call([]) === "[object Array]";

## 第四章 函数
+ 函数表达式其实也可以给函数命名。如：
<pre>var a = function b(){};</pre>这种做法的好处就是，a对应的不是一个匿名函数，在函数内部可以取到函数名（对于匿名函数报错的提示也有好处，可以定位到函数名），当然取和_a_不一样的函数名比如上面的_b_是不推荐的，因为其在ie下面实现得有问题。
+ 初始化时分支（Init-time branching）
  - 这种模式是一种优化模式，当知道某个条件在整个程序生命周期内都不会发生改变时，仅对该条件测试一次是很有意义的。如：
  <pre> var utils = {};
if(window.addEventListener) {
   utils.addListener = function(el, type, fn) {
       el.addEventListener(type, fn, false);
   }
}
else {
	utils.addListener = function(el, type, fn) {
       el.attachEvent('on' + type, fn);
   }
}
</pre>上述做法的好处就是只需要初始化一次之后不需要再次判断当前浏览器执行环境。

+ 备忘模式
	- 其实就是通过缓存函数结果，让重复调用的函数从缓存读结果来提升性能，没啥好说的。
+ curry化函数
	- curry话是指部分调用函数返回新函数的现象，比如著名的add(5,4)和add(5)(4)，直接上通用的curry化方法:
	  <pre>
	  function curry(fn) {
	     var storeArgs = Array.prototype.slice(arguments); //缓存调用参数
	     return function() {
		     var newArgs = storeArgs.concat(Array.prototype.slice(arguments));
			 return fn.apply(null,args); 
	     } 
	  }
	  function add(a, b, c, d, e) {
	       return a + b + c + d + e;	
	  }
	  curry(add ,1 , 2)(3, 4, 5);
	  </pre>

##第五章 对象创建模式
+ 通过命名空间函数
	- 大意就是通过 对象字面量 {} 来管理全局变量
	- 这里有一种写法是
	<pre>
		var obj = obj || {}; //如果没有obj再创建新的命名空间
		//这是一个很常见的写法，只不过看到这个代码，突然想起比如已经有了obj = 1;
		//那么下面这个 var obj; 会不会让obj = undefined;影响后面的取值？
		//上述代码应该可以拆分成： var obj; obj = obj || {}; 
		//obj会受var的影响吗？看下面两段代码
		var obj = 1; var obj; console.log(obj);//1
		var obj = 1; var obj = undefined; console.log(obj);//undefined
  		//可以看出 var obj;和 var obj = undefined;还是有区别的，区别就是后者有显示的赋值过程
		//在ES3规范中， 代码执行前会有初始化上下文和变量实例化两个过程，虽然在变量实例化时候var obj;会被默认赋上undefined，但是执行代码var obj = 1；变量obj值为1，下面一句var obj在执行的时候会被直接跳过，所以不影响后面的obj的值。
	</pre>
+ 私有成员
	- 通过闭包来实现私有成员
+ 将私有方法揭示为公有方法
	- 说白了就是备份一个方法，防止一个方法挂了导致全部都不能用了。
+ 模块化
	- 最早的模块化概念，类似YUI，var a.b = (function(){return {xx:xx}})(),只将一个模块API暴露出来。
	- 沙箱模式，用于解决越来越长的命名空间，不再依赖全局变量，而是提供一个沙箱环境，用于执行各个模块，但是又不会相互影响，其实是commonJS这种东西的另一种实现，就是通过API去约束。

# 第六章 代码复用模式
##继承
+ 经典继承
	- 经典继承就是很通用的通过构造函数继承父类的私有方法和属性，通过原型链继承父有的公用方法。
	<pre>
	function Parent(name) {
		this.name = name;
	}
	Parent.prototype.say = function() {
		console.log(this.name);
	}
	function Child(name) {
		Parent.apply(this, arguments);
	}
	Child.prototype = new Parent();//第一次调用parent()
	var instace = new Child('leon');//第二次调用parent()
	</pre>
	- 上述方法看起来很完美，但是也有缺点：**父构造函数** 会被调用两次，**私有属性**会被设置两次，对应上面的parent()和name属性，name属性会在child实例里面有一个以及其原型上有一个。原型上面的属性其实是冗余的，两次调用也是冗余的，这就会带来额外的性能开销。
+ 共享原型
	- `Child.prototype = Parent.prototype;` 好处是很便捷而且没有额外的开销，坏处非常明显，一处原型修改，全部都修改了。
+ 临时构造函数
	- 由经典继承的问题我们思考，其实关键问题在于`Child.prototype = new Parent();`这一句上面，我们希望子类能够连接上父类的原型，但是简单的通过将子类的原型变成父类的一个实例的话又会让父类白调用一次构造函数，而通过共享原型又会导致原型指向同一个，所以有一种方式就是借用一个空函数来**传递原型**：
	<pre>
	function inherit(C, P) {
		var F = function(){};
		F.prototype = P.prototype;
		C.prototype = new F();
		C.supperClass = P.prototype;
		C.prototype.constructor = C;
	}
	</pre>
	- 这就是一种最常用的继承方式
## Kclass
Kclass是对上面的一种总结：
<pre>
//假设语法糖kclass使用方法如下：
var Man = Kclass(null, {
	__construct: function (what) {
		console.log("Man is constructor");
		this.name = what;
	},
	getName: function () {
		return this.name;
	}
});
var SuperMan = Kclass(Man, {
	__construct: function () {
		console.log("SuperMan is constructor");
	},
	getName: function () {
		var name = SuperMan.uber.getName.call(this);
		return "I am" + name;
	}
});
//根据设计实现方式如下：
function Kcalss(Parent, props) {
	var Child, F, i;
	//构造函数
	Child = function () {
		if （Child.uber && Child.uber.hasOwnProperty("__construct")） {
			Child.uber.__construct.apply(this, arguments);
		}
		if (Child.prototype.hasOwnProperty("__construct")) {
			Child.prototype.__construct.apply(this, arguments);
		}
	}
	//继承
	Parent = Parent || Object;
	F = function () {};
	F.prototype = Parent.prototype;
	Child.prototype = new F();
	Child.uber = Parent.prtotype;
	Child.prototype.constructor = Child;
	//属性
	for (i in props) {
		if (props.hasOwnProperty(i)) {
			Child.prototype[i] = props[i];
		}
	}
	return Child;
}
</pre>