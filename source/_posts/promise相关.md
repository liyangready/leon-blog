title: "promise相关"
date: 2015-08-10 16:00:05
tags: promise
---
# 基本使用

*以ES6的 promise 为准。*   
在处理一个异步操作时，如果希望通过promise来实现，我们只需要两步：  

+ 封装一个promise对象
	<pre>
	function someSyncFn() {
	  return new Promise(function(resolve, reject) {
	    setTimeout(function() {
          console.log("after 2s");
		  resolve(2000);
	    }, 2000);
	  })
	}
	</pre>
+ 调用then方法
	<pre>
	someSyncFn().then(function(value) {
		console.log("getValue:" + value);
	});
	</pre>

单从上述方法其实并不能看出来promise的优势，一个callback改成了一个then而已，我们看多个异步操作的情况:

<pre>
  function someSyncFn1() {
	  return new Promise(function(resolve, reject) {
	    setTimeout(function() {
		  console.log("after 2s");
		  resolve(2000);
	    }, 2000);
	  })
  }
  function someSyncFn2(value1) {
	  return new Promise(function(resolve, reject) {
	    setTimeout(function() {
              console.log("after 5s");
              resolve(3000 + value1);
	    }, 3000);
	  })
  }
  someSyncFn1().then(someSyncFn2).then(function(value) {
	console.log("getValue:" + value);
  });
</pre>
我们发现，当链式异步并且存在先后顺序的时候，通过promise的确能够写出更简洁的代码，而且并不需要很多api，和单个异步一样，只需要两步，封装promise对象和调用then方法。

对于平行的异步调用，promise提供了 promise.all
<pre>
function someSyncFn1() {
	  return new Promise(function(resolve, reject) {
	    setTimeout(function() {
		  console.log("after 2s");
		  resolve(2000);
	    }, 2000);
	  })
  }
  function someSyncFn2() {
	  return new Promise(function(resolve, reject) {
	    setTimeout(function() {
              console.log("after 5s");
              resolve(3000);
	    }, 3000);
	  })
  }
Promise.all([someSyncFn1(), someSyncFn2()]).then(function(results) {
	console.log(results);
});
</pre>
上面的 results会顺序保存 两个异步的结果。

# 问题

从上面的基本用法，我们可能会觉得 promise 非常简单，但是我经常看到各种奇怪的写法，有好的也有反模式，我发现并不能弄懂它们的区别。比如：

+ then中直接调用方法
<pre>
 
  function someSyncFn1() {
	  return new Promise(function(resolve, reject) {
	    setTimeout(function() {
		  console.log("after 2s");
		  resolve(2000);
	    }, 2000);
	  })
  }
  function someSyncFn2(value1) {
	  return new Promise(function(resolve, reject) {
	    setTimeout(function() {
              console.log("after 5s");
              resolve(3000 + value1);
	    }, 3000);
	  })
  }
  someSyncFn1().then(function(value) {
	someSyncFn2(value); // 在then中直接调用 第二个异步函数
  }).then(function(value) {
	console.log("getValue:" + value);
  });
</pre>

 + then中返回一个值

<pre>
  function someSyncFn1() {
	  return new Promise(function(resolve, reject) {
	    setTimeout(function() {
		  console.log("after 2s");
		  resolve(2000);
	    }, 2000);
	  })
  }

  someSyncFn1().then(function(value) {
    console.log('async console');
    return value; //在then中返回值
  }).then(function(value) {
	console.log("getValue:" + value);
  });
</pre>

+ 一个异步之后需要多个回调的两种写法的区别：
<pre>
 //写法1
 someSyncFn1().then(callback1).then(callback2); 
 //写法2
 var promiseRt = someSyncFn1();
 promiseRt.then(callback1);
 promiseRt.then(callback2); 

 //是否写法1 是串行，写法2是 并行呢？ 
</pre>

+ 以及更经典的，下面四种写法区别在哪里：
<pre>
doSomething().then(function () {
    return doSomethingElse();
})；

doSomethin().then(functiuoin () {
    doSomethingElse();
});

doSomething().then(doSomethingElse());

doSomething().then(doSomethingElse);
</pre>

我们发现上面的执行结果乱七八糟，如果不能了解 promise内部执行流程，很难回答上面的问题，于是我们看一下官方规范到底是怎样的。
# promiseA+规范

promiseA+规范从规范的角度解释了promise的行为，其规范并不长，由于已经有一篇质量不错的翻译，这里就直接贴上引用：   
[Promise/A+规范](http://segmentfault.com/a/1190000002452115)

在规范中值得关注的几点：


> then 必须返回一个promise.

> `promise2 = promise1.then(onFulfilled, onRejected);`

> + 如果onFulfilled 或 onRejected 返回了值x, 则执行Promise 解析流程[[Resolve]](promise2, x).
> + 如果onFulfilled 或 onRejected抛出了异常e, 则promise2应当以e为reason被拒绝。
> + 如果 onFulfilled 不是一个函数且promise1已经fulfilled，则promise2必须以promise1的值fulfilled.
> + 如果 OnReject 不是一个函数且promise1已经rejected, 则promise2必须以相同的reason被拒绝.

以及[解析流程](http://segmentfault.com/a/1190000002452115#articleHeader4)

由此可见，不管给promise.then()传递怎样的实参，最终都会返回一个新的promise对象，而接下来如果链式调用then方法实际上是绑定在了这个新生成的promise对象上面。     
我们日常书写的存在疑虑的部分大都是在 `then` 的返回值上面，可能会有几种写法：

+ return undefined或者一个值
+ return 一个promise
+ throw 一个error

return undefined或者一个值的时候，新的promise对象将会以x为值 fulfill 从而直接触发了接下来的then方法并且把 这个值传递过去。

return 一个promise后，新生成的promise2将会等待这个返回的promise状态从 pending-resolve或者reject的过程，以它的状态来 fulfill或者rejects 自己的回调。 如果promise已经成功或者失败了，它也直接使用return的promise的值来触发自己的then方法。

# 分析
对于之前的问题：

1 then中直接调用方法，不管这个方法是同步的还是异步的，只要其没有显示的使用return，js默认return了undefined，按照上述规范描述，将会直接触发接下来的then方法，并且将undefined作为值传递下去。

2 then中return了一个值，同1.

3 写法1和写法2都是并行，因为写法1属于返回一个 undefined的情况，链式then方法都会被执行，并不会等待。另外，写法1的then方法是挂在不同的promise对象上，写法2是挂在同一个上面。

4 [谈谈使用 promise 时候的一些反模式](http://efe.baidu.com/blog/promises-anti-pattern/?utm_source=tuicool)

# 进阶

