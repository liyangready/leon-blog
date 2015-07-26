layout: title
title: "把书看薄-javascript模式-设计模式"
date: 2015-03-29 22:57:16
tags:
---
#第七章 设计模式
## 单体模式
最常见的单体其实就是一个对象字面量: `var obj = {"key": "value"};`即使有一个对象字面量拥有相同的属性`var obj2 = {"key": "value"};`判断`obj === obj2` 也会返回false；

还可以通过静态成员或者私有属性来实现单例，即当new 操作执行的时候先判断有没有创建过实例，如果有直接返回。

单体模式除了对象字面量经常使用外，在开发中不太常见。
<!--more-->
##工厂模式

工厂模式的目的是为了创建对象。

其最明显的特征是一般不用使用new来创建，而是 XXX.factory("str");  一个汽车工厂的demo：

<pre>
function CarMaker() {}
CarMaker.prototype.drive = function () {
  return "Vroom, I have" + this.doors + "doors";
}
CarMaker.factory = function (type) {
  var constr = type, newcar;
  if (typeof CarMaker[constr] !== "function") {
    throw {
       name: "Error",
       message: constr + " dosen't extis"
    }
  }
  if (typeof CarMaker[constr].prototype.drive !== "function") {
    CarMaker[constr].prototype = new CarMaker();
  }
  newcar = new CarMaker[constr]();
  return newcar;
}
CarMaker.Compact = function () {
  this.doors = 4;
}

var corolla = CarMaker.factory("Compact");
</pre>

工厂模式在日常开发中的应用场景主要用来创建相似对象，从而避免重复工作。和 **普通继承** 比起来，普通继承可以继承公用方法，但是构造函数还是需要自己声明。但是工厂模式连构造函数都不需要自己声明。当然它和 继承 是两码事，继承是为了重载某些共有方法，工厂模式可能大部分共有方法都是一样的，只是具有不同的私有属性。

##迭代器模式

迭代器模式其实就是一个复杂的 for 循环，用于遍历某种数据结构，当然它可以比for循环灵活的多，比如可以提供方法从头开始、跳步执行等等。调用的时候通过 Obj.next() 去通知计步器向下：

<pre>
var agg = (function () {
  var index = 0, data = [1, 2, 3, 4, 5], length = data.length;

  return {
    next: function () {
      var element;
      if (!this.hasNext()) {
        return null;
      }

      element = data[index];
      index = index + 2;
      return index;
    },

    hasNext: function () {
      return index < length;
    }
  }
}());


</pre>


##装饰者模式/门面模式

装饰者模式是指在通过从预定义的装饰者对象中添加功能，从而在运行时调整对象。

就是说去动态的更改某个方法以让它适合新的场景。

e.g :

<pre>
var sale = new Sale(100); //某个售价为100的商品
sale = sale.decorate('fedtax'); //增加联邦税
sale = sale.decorate('rmb'); //转换成rmb

sale.getPrice(); // rmb 780
</pre>

从上述例子可以看出来，装饰者模式可以让某个固定的方法变得非常灵活。比如上述的`getPrice();`

两种实现方法，第一种方法，把每一种装饰者都当作一个对象，或者说一个中间件，层层继承，先调用rmb对象的getPrice，然后调用父级别，一直往上，拿到最原始的价格。

<pre>
function Sale(price) {
  this.price = price || 100;
}
Sale.prototype.getPrice = function () {
  return this.price;
}
Sale.decorators = {};

//fedtax
Sale.decorators.fedtax = {
  getPrice: function () {
    var price = this.uber.getPrice();
    price += price * 5 / 100;
    return price;
  }
}

//rmb
Sale.decorators.rmb = {
  getPrice: function () {
    var price = this.uber.getPrice();
    ...
    return price;
  }
}

//decorators
Sale.prototype.decorators = function () {
  var F = function () {},
      overrodes = this.constructor.decorators[decorator],
      i, newobj;
  F.prototype = this;
  newobj = new F();
  newobj.uber = F.prototype;
  for (i in overrides) {
    if (overrides.hasOwnProperty[i]) {
      newobj[i] = overrides[i];
    }
  }
  retutn newobj;
}
</pre> 		

上述方法是通过原型链的链式调用实现，先方法更简单，通过列表实现：

<pre>
function Sale(price) {
  this.price = price || 100;
  this.decorators_list = [];
}
Sale.decorators = {};

//fedtax
Sale.decorators.fedtax = {
  getPrice: function (price) {
    price += price * 5 / 100;
    return price;
  }
}

//rmb
Sale.decorators.rmb = {
  getPrice: function (price) {
    ...
    return price;
  }
}
//price通过参数 传给不同的门面
//decorate
Sale.prototype.decorate = function (decorator) {
  this.decorators_list.push(decorators);
};

Sale.prototype.getPrice = function () {
  var price = this.price,
      i,
      length = this.decorators_list.length,
      name;
  for (i = 0;i < max; i += 1) {
    name = this.decorators_list[i];
    price = Sale.decorators[name].getPrice(price);
  }
  return price;
}

</pre>

##策略模式

策略模式就像是一个适配器，同一个工作接口，却能根据客户正在试图执行任务的上下文来选择指定算法。

比如解决表单验证，一个validate()方法，无论表单是什么类型，都能支持校验。
<pre>
var validator = {
  type: {}, //所有可用的检查方法
  config: {}, //当前要检查哪些
  message: [],//对应的结果
  validate: function (data) {
	//data: {}要检验的对象，key和config一致
    ...
    for (i in data) {
      if (data.hasOwnProperty(i)) {
        type = this.config[i];
        checker = this.type[type];
        result = cheker.validate(data[i]);
      }
      ...
    }
  }

}
</pre>
##外观模式

外观模式非常简单，就是一种封装，非常常见，把一些不同的业务封装成统一接口，比如:

<pre>
function stop (e) {
  e.preventDefault();
  e.stopPropagation();
}

</pre>

##代理模式

代理模式中，一个对象充当另一个对象的接口。一般用作合并某些复杂的开销统一进行，它和外观模式的区别是，外观模式是封装多个方法统一调用为了方便，而代理则是在使用者和提供者之间提供了一层过滤，作为中间者。

书上的例子是一个checkbox，每个checkbox被点击的时候会发一次ajax请求，然后展开，中间放一层代理的好处是可以设置一个延时，比如某个用户快速点击了3个checkbox，传统模式就是需要发送3次ajax请求，而中间加了一层代理，所有的请求由代理发出，代理可以设置比如2s延时，如果2s内有新的勾选，就先等所有勾选全部组合好再去发送ajax。

##中介者模式

中介者模式用于解耦，有些对象之间耦合很紧，一个对象会影响另一个对象，通过一个中介来在他们之间通信，其实和代理模式有点像，代理某种意义上来讲也是一种中介，但是中介应用范围更广，它可以用于状态的维护，书中的例子：

一个按键游戏，玩家一按2，玩家二按0，计分板看谁按得快。

中介者一方面负责接收玩家一和玩家二的消息，另一方面负责维护计分板，还要监听键盘事件等等，而玩家和计分板就被解耦了，从而玩家数量的拓展和计分板的拓展都会很容易，涉及到的改动只有中介者，不会互相影响。


##观察者模式

多用于异步处理。

观察者模式就不多说了，太常见了，所有的dom和外设之间的交互：比如鼠标事件、键盘事件，还要node中的事件模块都大量使用了观察者模式，还有最近很火的promise对象，也是使用了这种模式。

实现原理其实就是维护一个有序队列，在事件触发的时候推送给队列里面的所有订阅者。