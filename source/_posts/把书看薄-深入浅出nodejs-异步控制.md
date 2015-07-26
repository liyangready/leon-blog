layout: title
title: "把书看薄-深入浅出nodejs-异步控制"
date: 2015-04-12 18:52:54
tags: 读书笔记
---

#第4章 异步编程
异步编程对于前端之重要性不言而喻，而 _深入浅出node.js_书中这一章也写得非常精彩，所以必须得单拿出来做笔记。
## 异步编程的难点
+ 异常处理，try catch无法获取到异步的代码
+ 函数嵌套过深
+ 阻塞代码
+ 多线程编程
## 异步编程解决方案

<!--more-->

+ 事件发布/订阅模式
	- node的事件来源于 _events_模块，类似于dom里面的事件，具有 on/addListener、once、removeListener、emit等等方法，如：
	<pre>
	//订阅
	emitter.on("event1", function (message) {
		console.log(message);
	});
	//发布
	emitter.emit("event1", "I am message!");
	</pre>
	- 有两点值得注意的地方：
	
		1 如果一个事件超过10个侦听器，将会得到一条警告。调用 emitter.setMaxListeners(0);可以将这个限制去掉。
		2 如果在运行期间的错误触发了error事件，EventEmitter会检查是否对error事件添加过监听，如果监听了，交给监听器处理，否则错误作为异常抛出。
	- 解决雪崩问题
	雪崩问题是指高并发缓存失效的场景。
	<pre>
	var select = function (callback) {
		db.select("SQL", function (results) {
			callback(results);
		});
	}
	//如果站点刚刚启动，没有缓存，上面又没有锁，并发情况下数据库会反复查询
	//改进一 加个锁
	var status = "ready";
	var select = function (callback) {
	    if (status === "ready") {
	        status = "pending";
	        db.select("SQL", function (results) {
	            status = "ready";
	            callback(results);
	        });
	    }
	}
	//上述问题在于，并发过来的请求只有一个会返回数据，其他的无法得到。
	//继续改进
	var proxy = new events.EventEmitter();
    var status = "ready";
    var select = function (callback) {
        proxy.once("selected", callback);
        if (status === "ready") {
            status = "pending";
            db.select("SQL", function (results) {
                proxy.emit("selected", results);
                status = "ready";
	        });
        }
    }
	</pre>
	- 通过哨兵变量来处理监听多异步处理情况。说白了其实就是每个异步操作完成了调用一下固定函数，count++，只有count = 固定值的时候，才处理最后的回调。

+ Promise
	- promiseA+规范已经被集成到ES6里面，可以看一篇我小伙伴的译文 [promiseA+规范](http://www.ituring.com.cn/article/66566) 
	- 深入浅出里面着重介绍了 promise/A,也就是promise/deffered模式。也实现了一版简单的：
	<pre>
	var Promise = function () {
      EventEmitter.call(this);
    }
	Promise.prototype.then = function (fullfilledHandler, errorHandler, progressHandler) {
      if (typeof fullfilledHandler === 'function') {
        this.once('success', fullfilledHandler);    
      }
      if (typeof errorHandler === "function") {
        this.on("error", errorHandler);
      }
      if (typeof progressHandler === "function") {
        this.on("progress", progressHandler);
      }
      return this;
    }
    var Deffered = function () {
      this.state = "unfulfilled";
      this.promise = new Promise();
    }
    Deffered.prorype.resolve = function (obj) {
      this.state = "fulfilled";
      this.promise.emit("success", obj);
    }
    Deffered.prorype.reject = function (obj) {
      this.state = "failed";
      this.promise.emit("error", err);
    }
	</pre>
    其实弄懂了原理promise也是非常简单，promise对象维护了3个状态，然后通过then方法注册成功失败事件，在deffer对象里面去触发，最本质的原理其实还是通过事件机制实现。

    有一个问题是上面明显一个对象就能搞定，为什么还要引入deffer对象？ 先看使用上面：

    <pre>
    var promisefy = function () {
      var deffered = new Deffered();
      var result = "";    
      res.on("data", function (chunk) {
        results += chunk;
        deffered.progress(chunk);
      });
      
      res.on("end", function () {
        deffered.resolve(result);
      });

      res.on("error", function (err) {
        deffered.reject(err);
      });
      return deffered.promise;
    }
    //调用
    promisfy(res).then(function (){
    //done
    }, function () {
    //err
    });
    </pre>
    从上述代码可以看出来：
    
    1 promisefy用作在异步函数里面注入固定的代码 resolve/reject ，返回promise对象。

    2 promise对象注册事件，从而实现1步骤里面的异步完成时来触发这些事件。

    3 为什么要把deffer和promise分开，是因为 除了封装在promisefy里面的deffer对象，promise对象是拿不到resolve和reject事件的，从而保证状态不被外界修改，promise只能用作注册函数，而触发必须在回调里面用deffer对象触发，**这样设计是出于安全性考虑，promisefy调用之后返回的promise对象保证只能绑定不能修改内部的状态**。

    4 想想这种模式比事件订阅模式高级的地方就是 不用自己去维护那个 done 方法，虽然还需要自己写promisefy方法，但是这个方法完全可以抽象出来。用指定的API来保证方法调用，比如ES6里面的 Promise方法，就是一个通用的 promisefy方法。

    5 实现多异步操作：
    <pre>
    Deffered.prototype.all = function (promise) {
       var count = promise.length;
       var that = this;
       promise.forEach(function (promise, i) {
         promise.then(function (data) {
           count--;
           result[i] = data;
           that.resolve(results);
         });
       })
    }
    </pre>
    其实就是维护了一个promise队列，当全部都完成时再调用最后的回调。

    6 链式调用：
    <pre>
    //实现链式调用就是在promise对象上面维护一个按顺序的queue，调用then的时候按顺序压入队列，每次resolve改成拿队头的回调函数出来执行。之前的promise.then是绑定的once方法，现在其实就不需要事件机制了，维护事件队列就行了。 
    </pre>