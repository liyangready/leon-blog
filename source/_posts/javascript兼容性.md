layout: title
title: "javascript兼容性"
date: 2015-04-16 11:18:58
tags: 兼容性
---
### ecmascipt
+ **IE6-8** 函数声明和函数表达式
  - **ie6-8**函数名标识符里面可以有 `.` , 标准浏览器会报错,如下代码片段一
  - **ie6-8**函数表达式标识符key被外部访问，标准浏览器只能用于内部访问，如下代码片段二
  <pre>
  //标准报错
  function A(){}
  function A.prototype.b(){}
  var a=new A();
  alert(typeof a.b);

  //ie6-8能取到，标准浏览器不行
  var a=function b(){};
  alert(typeof b);  
  </pre>
  
 **解决方法**：不要使用带`.`的函数名标识符，函数表达式标识符不要作为引用的目的出现，出现在标识错误匿名函数等。

+ **Firefox对条件语句块内部函数声明处理有差异** 
<pre>
//其它浏览器不管 true或者false，都是2，firefox是1
//规范定义是函数声明之后出现在全局环境或者函数体里面，在判断块里面当作是在函数里面，所以后面的会覆盖前面的
function foo(){
  if(true){
    function bar(){alert(1);}
  }
  else{
    function bar(){alert(2);}
  }
  bar();
foo();
</pre>

+ **各浏览器中 Date 对象的 getYear 方法的返回值不一致**          
根据规范，这个方法将返回当前时间的年份值与 1900 的差值，如 1800 年返回 -100，2010 返回 110。但 IE 仅在一个 1900 - 1999 年之间的日期值上调用 getYear 方法时，减去 1900，在其他的日期中返回的是实际的年份，就和 getFullYear 一样。           
**解决办法**
使用Date.prototype.getFullYear() 代替
+ **ie6-8**不支持JSON对象及其方法
+ **Chrome Opera 中 for-in 语句遍历出对象属性的顺序与定义的不同**，不要依赖key的排序
+ **IE6-8**不能在 JSON 字符串或对象直接量的最后一个键值对后加 ','
+ **IE6**不能使用下标来访问字符串
+ [各浏览器中用 for in 可以遍历对象中被更新的内置方法存在差异](http://w3help.org/zh-cn/causes/SJ5003)
+ [IE6，IE7，IE8不会忽略数组直接量的末尾空元素](http://w3help.org/zh-cn/causes/SJ2007)
+ [元素的内联事件处理函数的特殊作用域在各浏览器中存在差异](http://w3help.org/zh-cn/causes/)
+ [Array.prototype.sort当使用了 comparefn 后返回值不为 -1、0、1时，各引擎实现排序结果不一致](http://w3help.org/zh-cn/causes/SJ9013)


##dom
+ IE6 IE7 IE8(Q) 中的 getElementById 方法的参数不区分大小写
+ IE 在创建 DOM 树时会忽略某些空白字符
