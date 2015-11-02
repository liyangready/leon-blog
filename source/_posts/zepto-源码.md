title: "zepto 源码"
date: 2014-10-30 15:08:55
tags: 源码
---

$ = Zepto;    
$是一个方法，方法上面可以挂很多私有属性，比如 ajax这种，这些私有属性可以通过 $.xxx 使用，对应zepto对象 $().xx不可访问。   
$() 会执行 $.zepto.init 方法，这个方法返回的是 $.zepto.Z() , 而Z是zepto的一个私有构造函数， Z() 就能返回一个实例，也就是 zepto实例，所以$() 会返回不同的Z实例。实例上面也有方法，共享的方法当然是挂在Z.prototype上面，`zepto.Z.prototype = Z.prototype = $.fn` 可以看出来，操作 $.fn其实就是操作 Z的prototype。可以通过它来拓展 zepto。  

zepto的选择器由于不用兼容ie6，多半采用的是dom level3的querySelectorAll， 除了对 id和tag选择器做了优化直接使用 getElementById 和 getElementsByTagName。  

zepto事件，delegate、live等都是on的语法糖，on方法主要在 zepto.event里面的 add方法中，其中会包裹下回调方法，如果是dom节点，会直接调用addEventListener来绑定事件，比如使用return false可以直接实现 `e.preventDefault(), e.stopPropagation()`等，还会模拟mouseenter和mouseleave，同时会把element和事件回调缓存起来。     
trigger，如果是element，有新的 `dispatchEvent` 方法，zepto会模拟一个event对象，然后触发事件，否则就会去之前缓存的 handler中找对应的事件来触发。

zepto ajax没啥说的，主要是 `  if ((xhr.status >= 200 && xhr.status < 300) || xhr.status == 304 || (xhr.status == 0 && protocol == 'file:')) {`判断成功的方法是这些。然后就是对于jsonp的eval，这么使用的 `if (dataType == 'script')    (1,eval)(result)` 为什么要这么用，有一个参考链接介绍全局 eval和innter eval的区别     
http://perfectionkills.com/global-eval-what-are-the-options/

zepto touch,通过 touchstart touchend 和touchmove来模拟一些事件， >750ms longTap， 还有tap、swipe等。

zepto animate主要通过transition和css3 animation来实现
