title: "dom兼容性问题收集"
date: 2015-08-07 16:26:01
tags: "兼容性"
---

## 位置相关

+ 页面滚动距离（scrollTop/scrollLeft）：

IE9+ & WEBKIT: `window.pageYOffset == window.scrollY;`  
兼容写法 : 
<pre>
var supportPageOffset = window.pageXOffset !== undefined;
var isCSS1Compat = ((document.compatMode || "") === "CSS1Compat");
var x = supportPageOffset ? window.pageXOffset : isCSS1Compat ? document.documentElement.scrollLeft : document.body.scrollLeft;
var y = supportPageOffset ? window.pageYOffset : isCSS1Compat ? document.documentElement.scrollTop : document.body.scrollTop;
</pre>
`document.compatMode`表示是否是标准文档模式。

+ 窗口大小

IE9+ & webkit:  `window.innerHeight`表示浏览器网页部分可视区域高度。`window.outerHeight`表示浏览器高度。

兼容写法： 
<pre>
var h = window.innerHeight
|| document.documentElement.clientHeight
|| document.body.clientHeight;
</pre>

window.outerHeight 一般来说等于 `window.screen.availHeight`
包含了浏览器工具条的高度，而`window.screen.height`是指屏幕的高度。

+ 各种高度

一篇详细的测试文章： [offsetHeight, clientHeight与scrollHeight的区别](http://blog.csdn.net/woxueliuyun/article/details/8638427)
