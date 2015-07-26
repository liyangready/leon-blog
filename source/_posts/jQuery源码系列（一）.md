layout: title
title: "jQuery源码系列（一）"
date: 2015-03-27 11:43:26
tags: 源码
---

#start - (jquery1.8.3)

##core.js

+ 开头很多缓存变量是一个类库需要学会的
<pre>
document = window.document,
location = window.location,
navigator = window.navigator,
core_push = Array.prototype.push,
core_slice = Array.prototype.slice,
core_indexOf = Array.prototype.indexOf,
core_toString = Object.prototype.toString,
core_hasOwn = Object.prototype.hasOwnProperty,
core_trim = String.prototype.trim,
...
//类似这种就可以大大减少后续使用时候对变量查询和对对象属性的查询

</pre>
<!-- more -->
+ jQuery构造函数
	- jQuery是个构造函数，但是它不产生实例，jQuery很奇葩的在构造函数中调用了其它的构造函数，而这个构造函数还是放在jQuery的prototype里面，**原因还在思考，有大神可以指教**。能想到的一个原因就是可以省去new来使用构造函数，当然省去new一般都用 instanceof来判断，不知道jquery这么玩是为什么。。。所以说它奇葩
	<pre>
	// Define a local copy of jQuery
	jQuery = function( selector, context ) {
		// The jQuery object is actually just the init constructor 'enhanced'
		return new jQuery.fn.init( selector, context, rootjQuery );
	}
	//可以看出来 如果new $或者直接$()，返回的是init的实例，我们看这个init
	jQuery.fn = jQuery.prototype = {
		constructor: jQuery,
		init: function() {}
	}
	jQuery.fn.init.prototype = jQuery.fn;

	</pre>
	- 看出来了没，`jQuery.fn`就是`jQuery.prorotype`，jQuery构造函数返回的是`jQuery.fn.init`的实例就是`jQuery.prototype.init`的实例，返回了自己原型上面一个方法的实例，如何保证这个实例能访问到原型上面的其它方法呢？`jQuery.fn.init.prototype = jQuery.fn;` 挺有意思的哈。
+ jQuery原型上面的方法
	- init，上面说了，这个init方法返回的是jQuery实例，就是我们通用的 $()方法的返回值。有下面几种情况：
	 <pre>
		// Handle $(""), $(null), $(undefined), $(false)
		if ( !selector ) { //不用解释
			return this;
		}
		//如果传进的参数是一个dom节点，返回一个类数组对象，也是经典jquery对象，
		//如 $(document) -> [document] 但是其不是数组有jquery原型上面的所有方法
		if ( selector.nodeType ) {
			this.context = this[0] = selector;
			this.length = 1;
			return this;
		}
		//html string
		if ( typeof selector === "string" ) {
			//1 html 如 $("<div></div>"),会创建新的dom对象/dom片段
			//2 function , $(function(){}),domready去执行
			//3 #id，id是直接拿出来 返回这样更快 
			//4 其它选择器,调用Sizzle的find方法去执行
		}	
	</pre>
	- 剩下的方法就不详细讲解了，挑有意思的讲
	- toArray , `Array.prototype.slice.call()`
	- pushStack , 这个方法是jQ内部维护了一个实例选择过的队列，有什么用？
	<pre>
		$("#parent").find("#son").xxxx ;//对parent下面的某个元素一堆操作之后如果想再对parent操作，jq有个end方法，可以回退到上一次的选择集合
		$("#parent").find("#son").xxxx.end();//神奇的回到了parent
		//就是因为它内部维护了一个队列，每个jq对象都有一个 prevObj的引用，指向上一次的集合。
		//pushStack就是为了实现这个队列。
		
	</pre>
	- sort、splice : `[].sort、[].splice` ，这个比 Array.prototype.sort省字节
+ jQuery.extend
extend实际上就是挂在jq原型上
	- extend方法：如果直接传入一个对象，相当于对jq扩展(**要注意的是 extend如果是扩展jq，调用jQuery.extend,如果是扩展jq的prototype，调用jQuery.fn.extend**)，jq的插件都是这种写法，如果传入多个对象，就是我们常用的用来合并几个对象，可以深拷贝和浅拷贝，深拷贝就是递归的过程。
+ jQuery.ready
	- $(fn) / $(document).ready(fn) , 用来在document ready的时候执行回调，ready会比load早不少。原理大概是这样的：
	<pre>
	jQuery.ready绑在jq对象上，作为回调。用户调用是$().ready ,这个ready方法是绑在jq的原型上面，它大概做下面几件事
	1 对于标准浏览器，监听 DOMContentLoaded
	2 对于ie ，监听 	onreadystatechange === "complete" ，以及 onload 以及
		setTimeout( doScrollCheck, 50 ); 这三者谁先完成都可以
	原型上面的ready回调就是jQuery.ready,它来保证只执行一次回调之类的工作
	</pre>