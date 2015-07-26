title: "前端工程构建之gulp"
date: 2015-03-17 18:03:11
tags: 前端工程
---
#前面的废话
一直在思考前端工程师的价值体现在哪里，或者说优秀前端工程师的竞争力在哪里？

+ 小公司喜欢什么样的人？
	- 能快速搞出东西来的人，使用各种工具、插件，分分钟起页面，切图切个三四天？怎么可能！管你什么ie6，能出东西就行了，最好再适应个移动端。。。
+ 大公司喜欢什么样的人？
	- 能写出可维护代码
	- 基本功扎实
	- 能够搞定ie6
	- *工程化/结构化/模块化，解决大规模前端代码问题*


很不巧，gulp这种东西，对于小公司能快速起页面，对于大公司能承担起工程化代码的作用，不得不用啊。当然各个逼格高一点的公司都会用自己的类gulp工具，比如百度的fis，我司的fekit等等，但是实际上从社区的角度来说，肯定无法与gulp比拟，在个人开发的时候，还是要选择gulp。

#grunt VS gulp
可以看这篇文章[Gulp vs Grunt](http://www.f2e.im/t/394)

#start
<!--more-->
gulp是典型的极简主义，导致其官方get start什么都没有。。。

+ 1 全局安装
<pre>npm install --global gulp</pre>
+ 2 在你的开发环境安装gulp
<pre>npm install --save-dev gulp
//这一步不能省，全局安装的gulp类似于一个命令行工具，在具体的业务中还需要引进gulp module
</pre>
+ 3 创建一个叫 gulpfile.js 的文件，内容如下
<pre>
var gulp = require('gulp');
gulp.task('default', function() {
  // place code for your default task here
});
</pre>

+ 4 执行 gulp 命令

 一个简单的应用就这么结束了，实际上 ，几乎所有应用都是这个套路。 gulp用起来就和写代码一样，你把需要的插件引进来，写进task，顺序执行就ok了。

# gulp API

+ gulp API设计也非常简单，就下面几个

##**gulp.src(globs[,options])**
<pre>
gulp.src('client/templates/*.jade')
  .pipe(jade())
  .pipe(minify())
  .pipe(gulp.dest('build/minified_templates'));
</pre>

src可以使用通配符匹配一个或多个，对于参数gulp是使用了开源的 _glob_ 来完成： [node-glob](https://github.com/isaacs/node-glob)
返回值是一种格式的流[Vinyl files](https://github.com/wearefractal/vinyl-fs)也开源了，感兴趣的可以了解一下，返回值就可以被pipe方法操作操作。

## **gulp.dest(path[,options])**
<pre>
gulp.src('./client/templates/*.jade')
  .pipe(jade())
  .pipe(gulp.dest('./build/templates'))
  .pipe(minify())
  .pipe(gulp.dest('./build/minified_templates'));
</pre>

看名字就知道，他和src是反的，类似于写的感觉，作为输出，可以输出到不同的文件夹，如果文件夹不存在就会建立新的文件夹。可以输出多个文件。

## **gulp.task(name[,deps], fn)**
<pre>
gulp.task('somename', function() {
  // Do stuff
});
</pre>

其中deps参数是一个数组，作为在你要执行的任务之前要执行的任务队列，它会先执行完数组里面的任务队列，再来执行你的任务。

同时，task可以支持异步，如果回调函数存在下面三种用法，就会实现异步：

**接受一个回调函数**
<pre>
// run a command in a shell
var exec = require('child_process').exec;
gulp.task('jekyll', function(cb) {
  // build Jekyll
  exec('jekyll build', function(err) {
    if (err) return cb(err); // return error
    cb(); // finished task
  });
});
</pre>

**返回了一个流**
<pre>
gulp.task('somename', function() {
  var stream = gulp.src('client/**/*.js')
    .pipe(minify())
    .pipe(gulp.dest('build'));
  return stream;
});
</pre>

**返回一个promise对象**
<pre>
var Q = require('q');

gulp.task('somename', function() {
  var deferred = Q.defer();

  // do async stuff
  setTimeout(function() {
    deferred.resolve();
  }, 1);

  return deferred.promise;
});
</pre>

上述是官方文档的描述，我们理解一下这几句话，什么叫可以是异步的？

跑下面这一段代码：
<pre>
var gulp = require('gulp');

gulp.task('test1' , function() {
	setTimeout(function(){
		console.log("test1 finish");
	},1000);
});

gulp.task('test2',function() {
	setTimeout(function(){
		console.log("test2 finish");
	},500);
});

gulp.task('default', ['test1','test2']);
</pre>

执行gulp，控制台输出：

<pre>
[19:19:12] Starting 'test1'...
[19:19:12] Finished 'test1' after 438 μs
[19:19:12] Starting 'test2'...
[19:19:12] Finished 'test2' after 86 μs
[19:19:12] Starting 'default'...
[19:19:12] Finished 'default' after 8.21 μs
test2 finish
test1 finish
</pre>

可以发现gulp默认是按顺序执行你的任务，但是它 **不会等待** ，test1虽然其实要在1s钟之后才会完毕，但是它按同步顺序执行，很快就结束了，然后开始test2，同样很快就结束了。
我们按上面的方法加入最简单的回调：


<pre>
var gulp = require('gulp');

gulp.task('test1' , function() {
	setTimeout(function(){
		console.log("test1 finish");
	},1000);
    cb();
});

gulp.task('test2',function() {
	setTimeout(function(){
		console.log("test2 finish");
	},500);
});

gulp.task('default', ['test1','test2']);
</pre>

</pre>

<pre>
[19:22:35] Starting 'test1'...
[19:22:35] Starting 'test2'...
[19:22:35] Finished 'test2' after 151 μs
test2 finish
test1 finish
[19:22:36] Finished 'test1' after 1.01 s
[19:22:36] Starting 'default'...
[19:22:36] Finished 'default' after 11 μs
</pre>
可以发现，控制台结果 test2 finish 和 test1 finish虽然没有变化，但是对于 **gulp** 来说，test1这个task是在1000ms之后才完成的，test2没有设置异步，所以很快完成。

有人会说了，这尼玛有啥用啊，反正task1和task2都会执行，而且执行顺序也不受影响。
是的，在上面这个例子中，你不期望task1和task2的顺序，的确不用callback去告诉gulp他们正确的执行顺序，假如你希望task1执行完了再执行task2，我们试下面这段代码。

<pre>
var gulp = require('gulp');

gulp.task('test1' , function() {
	setTimeout(function(){
		console.log("test1 finish");
	},1000);
});

gulp.task('test2',['test1'],function() {
	setTimeout(function(){
		console.log("test2 finish");
	},500);
});

gulp.task('default', ['test1','test2']);
</pre>

看console：
<pre>
[19:28:38] Starting 'test1'...
[19:28:38] Finished 'test1' after 592 μs
[19:28:38] Starting 'test2'...
[19:28:38] Finished 'test2' after 179 μs
[19:28:38] Starting 'default'...
[19:28:38] Finished 'default' after 8.21 μs
test2 finish
test1 finish
</pre>
坑爹了，我们按照官方定义的，给task2多传了一个参数告诉task2要在task1执行之后再去执行task1,没有问题，但是问题是test2 finish先输出，显然不对！
这就是因为你没有告诉gulp task1是异步的，不是马上就结束了，它不知道，然后傻叉的以为task1马上结束了，按你的吩咐去执行了task2，肯定不对撒，cb/stream/promise 就用在这儿，看我小改一下：
<pre>
var gulp = require('gulp');

gulp.task('test1' , function(cb) {
	setTimeout(function(){
		console.log("test1 finish");
		cb();
	},1000);
});

gulp.task('test2',['test1'],function() {
	setTimeout(function(){
		console.log("test2 finish");
	},500);
});

gulp.task('default', ['test1','test2']);
</pre>
看console：
<pre>
[19:33:21] Starting 'test1'...
test1 finish
[19:33:22] Finished 'test1' after 1.01 s
[19:33:22] Starting 'test2'...
[19:33:22] Finished 'test2' after 109 μs
[19:33:22] Starting 'default'...
[19:33:22] Finished 'default' after 9.03 μs
test2 finish
</pre>

果然完美了，说了要相信科学       

**从上面的例子可以看出来，当你需要顺序执行几个task的时候，第一，你需要在task2的参数上面声明需要先于它执行的task1，而且，如果task1是异步的，一定要记得告诉gulp，task1什么时候执行结束！**


##gulp.watch
 
gulp.watch(glob [, opts], tasks) or gulp.watch(glob [, opts, cb])

watch就是典型的hook，很好理解，当某个/些文件变动时，去执行某些命令或者执行回调:

<pre>
task用法
var watcher = gulp.watch('js/**/*.js', ['uglify','reload']);
watcher.on('change', function(event) {
  console.log('File ' + event.path + ' was ' + event.type + ', running tasks...');
});
cb用法：
gulp.watch('js/**/*.js', function(event) {
  console.log('File ' + event.path + ' was ' + event.type + ', running tasks...');
});
</pre>

很好理解就不解释了。

#结语
gulp很好很强大，上述只是官网的翻译，下一篇来一个完整的例子，gulp实战~~~