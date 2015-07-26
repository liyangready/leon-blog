layout: title
title: "jQuery源码系列（二）"
date: 2015-03-29 22:57:42
tags: 源码
---
#event.js
+ jq的事件处理核心都在**on**方法上面，无论你通过$("#xxx").click(); 或者通过bind/delegate/one 最后都是调用这个on方法。
##on
+ on方法处理了一大堆参数，形成语法糖，和业务太挂钩我们就不细看了，可以看到它最后会调用
<pre>
return this.each( function() {
       jQuery.event.add( this, types, fn, data, selector );
});
</pre>
<!-- more -->
所以我们去看 add方法：
## add
+ jQuery从1.2.3版本引入数据缓存系统，贯穿内部，为整个体系服务，事件体系也引入了这个缓存机制所以jQuery并没有将事件处理函数直接绑定到DOM元素上，而是通过.data存储在缓存.cahce上。
+ 第一步：获取数据缓存
<pre>
	//获取数据缓存
    elemData = data_priv.get( elem );
	//在$.cahce缓存中获取存储的事件句柄对象，如果没就新建elemData
</pre>
+ 第二步：创建编号
<pre>
	if ( !handler.guid ) {
        handler.guid = jQuery.guid++;
    }
</pre>
+ 后面的步骤可以参看 [on的详细操作](http://www.cnblogs.com/aaronjs/p/3444874.html) 