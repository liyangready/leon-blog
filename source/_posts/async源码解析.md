title: "async源码解析"
date: 2015-08-08 16:27:16
tags: 源码
---

## 前言

异步是javascript一个标志，尤其在nodejs中，操作数据库等往往带来大量的异步调用，所谓的“callback hell”由此而来，async作为成名较早的异步流程库，API设计属于异步1.0时代，其思想非常接近主流思考方向，源码必然很有参考价值，于是找时间看了一下其源码中的几个主要方法。

<!-- more -->
## async.parallel

<pre>
async.parallel = function (tasks, callback) {
  _parallel(async.eachOf, tasks, callback);
};
//这一步很简单，调用了 _parallel方法：
function _parallel(eachfn, tasks, callback) {
  //eachfn是async.eachOf
  //tasks是异步队列，callback就是平行异步的总回调
  callback = callback || noop;   
  var results = _isArrayLike(tasks) ? [] : {}; 
  //results保存着每个异步调用的返回结果
  //这一步循环调用tasks，我们看看eachfn是怎么定义的
 eachfn(tasks, function (task, key, callback) {
    。。。
  }, function (err) {
    callback(err, results);
  }


  async.eachOf = function (object, iterator, callback) {
    //经过once包装过的callback保证只执行一次，
    //考虑到同时执行的异步有出错的时候，callback不会多次执行
    callback = _once(callback || noop); 
    object = object || [];
    var size = _isArrayLike(object) ? object.length : _ke
    var completed = 0;
    if (!size) {
        return callback(null);
    }
    _each(object, function (value, key) {
        iterator(object[key], key, only_once(done));
    });
    function done(err) {
        if (err) {
            callback(err);
        }
        else {
            completed += 1;
            if (completed >= size) {
                callback(null);
            }
        }
    }
 };
 //eachof逻辑非常简单，其实就是传递一个数组或者对象给它，然后它会
 //遍历数组，执行传入的 iterator 处理方法，每执行一次+1，
 //直到全部执行完,调用最终的callback。它用来控制什么时候结束所有异步调用。
 
 //我们再回过头来看 iterator 即异步队列循环调用的时候给每个方法封装的包裹函数。
 
eachfn(tasks, function (task, key, callback) {
 //eachfn上面我们分析了，它就是用来控制结束的，所以 iterator 
 //也就是这个匿名函数必须调用它的done方法，它来做迭代器的控制判断
 //什么时候结束，这个done方法在这个匿名函数中就是 callback 参数。
 //什么时候调用callback是业务代码控制的，所以必须传给task
 //同时它再在这个done方法上面通过_restParam封装了一个方法，
 //用于保存结果到results
 //_restParam的方法是把业务代码的数据除了err以为都存到一个数组里面。
    task(_restParam(function (err, args) {
        if (args.length <= 1) {
            args = args[0];
        }
        results[key] = args;
        callback(err);
    }));
}, function (err) {
    callback(err, results);
});

</pre>

## async.waterfall

<pre>
async.waterfall = function (tasks, callback) {
    callback = _once(callback || noop);
    if (!_isArray(tasks)) {
        var err = new Error('First argument to waterfall must be an array
        return callback(err);
    }
    if (!tasks.length) {
        return callback();
    }
    function wrapIterator(iterator) {
        ...
    }
    wrapIterator(async.iterator(tasks))();

    //我们先看 async.iterator,这方法和 parallel 里面的eachof
    //有点神似，目的是把 tasks 包装成链状的，有一个next方法，指向下一个task。
    async.iterator = function (tasks) {
	    function makeCallback(index) {
	        function fn() {
	            if (tasks.length) {
	                tasks[index].apply(null, arguments);
	            }
	            return fn.next();
	        }
	        fn.next = function () {
	            return (index < tasks.length - 1) ? makeCallback(index + 1): null;
	        };
	        return fn;
	    }
	    return makeCallback(0);
	    };
    };
    //再回过头来看 wrapIterator 

   function wrapIterator(iterator) {
    //_restParam照样可以不用管，就是把一个方法的实参封装成err和一个数组
    return _restParam(function (err, args) {
        if (err) { 
            //错误处理，出错直接结束，因为没有调用next了
            callback.apply(null, [err].concat(args));
        }
        else {
            var next = iterator.next();
            if (next) {
				// 把下一个task作为最后一个形参传给当前的task
				// 业务代码结束调用 callback(err, data); 时：
                // callback就是下一个task。
                args.push(wrapIterator(next));
            }
            else {
                args.push(callback);
            }
            // 这个task的形参是由上一个task在业务代码中调用而来，最后一个固定是callback，前面为数据
            ensureAsync(iterator).apply(null, args);
        }
    });
  }
</pre>