# ECMAScript可执行程序和可执行上下文

### ES5可执行程序

+ 全局代码
+ eval代码
+ 函数代码

### ES5可执行上下文

当ECMAScript引擎执行一段ECMAScript程序的时候，进入不同的可执行程序会创建不同的可执行上下文。比如进入全局代码，会创建全局上下文，进入函数代码，会创建函数上下文。其中函数上下文最具有代表性，进入一段函数代码会发生下面的事情：

+ 引擎进入函数代码，**执行前**，建立**函数上下文**：
  
  - 声明this指向，如果未给函数执行传入this参数，this指向global（浏览器下面就是 window）。
    
  - 创建一个新的**词法环境**，将这个词法环境的**外部词法环境**指向函数的[[scope]]属性，而函数的[[scope]]属性是等于函数**创建时所处环境**的词法环境。
    
    在这里解释一下**词法环境**（Lexical Environments ）：
    
    词法环境是一个规范类型（实现规范的内部类型，不是给ECMAScript程序使用的）。词法环境包含两个内容—— **环境记录对象** 和 **外部词法环境**，其中环境记录保存着这个上下文中的变量存储情况，而外部词法环境连向外部的词法环境，在查找变量的适合形成**链式结构**，一步一步向上查找。
    
    解释一下**环境记录对象**：
    
    环境记录对象（Environment Records）有两种实现：声明式环境记录(*declarative environment records*,，下面简称为DER) 对象式环境记录 (*object environment records*, 下面简称为OER)。其中声明式环境记录会记录包括函数声明、var声明、catch语句等。对象式环境记录包括全局程序(window)和with语句。对象式环境记录可以理解为把变量等信息保存在自己的 *binding object* 上面，比如 window和with传入的object。 
    
    DER我们可以理解为保存着当前上下文所需要的变量，包括函数声明、var变量、arguments等。OER我们可以理解为保存着某个对象的变量，比如window对象和with使用的对象。
    
    ``` javascript
    //这一步我们创建了一个词法环境LE
    newLE = {
      DER: {
      	...//存储变量
      },
      outerLE: function.[[scope]] //指向外部词法环境形成链式结构
    }
    ```
  
  
  - 建立两个变量 `LexicalEnvironment` 和 `VariableEnvironment` 都指向上面说的 LE。
    
    ``` javascript
    LE = newLE; VE = newLE;
    ```
    
    这里为什么需要两个变量指向同一个词法环境对象我们稍后再说，现在先假设他们就是一个东西，这个里面存储了当前函数需要的变量。
  
+ 上下文创建完毕之后，发生**声明绑定实例化(Declaration Binding Instantiation)**
  
  声明绑定实例化就是把函数中的变量放在DER里面，放的顺序依次是 函数声明、形参和var变量。默认var变量都是 undefined
  
+ 代码执行
  
  代码执行阶段DER中存储的变量值会根据执行的代码被覆盖。其中涉及到**标示符解析**的过程。标示符解析就是变量查找的过程，它会找当前EC的LE，没有再找当前EC的outerEC的LE，一直向上便利。
  
  **LE和VE**的区别：
  
  + LE和VE在一开始都等于newLE，就是新创建的词法环境对象。
    
  + VE用作变量存储，再绑定声明实例化过程中，VE的变量被赋值和存储，LE在大部分时间都等于VE。
    
  + with语句、catch语句等词法环境种，LE会被赋新的值，从而LE != VE。
    
  + 所以LE是可变的，而VE从创建就一直指向同一个对象。
    
  + 变量查找是找LE，但是变量存储是存在VE上面的。在with语句中，会插入新的LE，让with语句中的变量先查找withObj，但是退出with语句，LE和VE的值又相等了，这就是为什么要设计LE和VE。因为有些情况下，**LE会被赋上新的值改变变量查找的情况**。
    
  + 函数声明解析的[[scope]] = 函数所处环境的VE；函数表达式解析的[[scope]] = 函数所处环境的LE；所以在with语句中，LE变化时，会影响到函数表达式的变量查找。并不会影响到函数声明的变量查找:
    
    ``` javascript
    //在控制台执行这两段脚本
    (function(){
     var a = 2;
     function test(){
          console.log(a)
     } 
     with({a:1}) { 
       test();
     }
    })();
    //因为with语句的LE会变成 {a:1},函数声明的[[scope]]是VE不受影响，所以第一个console会
    //输出2，但是下面那个LE会受影响，所以变成1
    (function(){
     var a = 2;
     with({a:1}) {
        var test = function(){
          console.log(a)
        }
       test();
     }
    })();
    ```



### 实例分析

``` javascript
//分析下面这段代码的执行过程
console.log(a);
console.log(b);
console.log(test);
var a = 1;
var b = function() {
  console.log(a);
}
function test(a) {
  console.log(a);
  function a() {};
  var a = function() {};
}
test(3);
b();
```

1 进入全局代码，创建全局上下文：

``` javascript
GEC = {
  this: window,
  VE: {
    OER: {}, //全局代码的环境记录是对象式的，binding object是window
    outer: null
  }
}
GEC.LE = GEC.VE;
```

2 全局代码声明绑定实例化：

``` javascript
GEC.VE = {
  OER: {
    a: undefined, //var a
    b: undefined,
    test: testFun //这一步还会将函数声明初始化，从而 testFun的 [[scope]] = GEC
  }
}
```

3 依次执行代码:

``` javascript
console.log(a);//查找GEC.LE.OER.a , 输出 undefined
console.log(b);//查找GEC.LE.OER.b , 输出 undefined
console.log(test);//查找GEC.LE.OER.test , 输出 function test
a = 1; // GEC.LE.OER.a = 1;
b = function...; //发现b会被赋值一个函数，这时候会初始化函数表达式，将b.[[scope]] = GEC
test(3); //进入test函数，所以会创建test函数上下文
```

+ 创建test函数上下文：
  
  ``` javascript
  testEC = {
    VE: {
      DER: {},
      outer: testEC.[[scope]] //testEC.[[scope]] = GEC 
    }
  }
  ```
  
+ test函数声明绑定实例化：
  
  ``` javascript
  testEC = {
    VE: { 
      DER{
       a: function a(){}, //根据函数表达式>形参>var 的优先级关系
      }
      outer: testEC.[[scope]] //testEC.[[scope]] = GEC 
    }
  }
  ```
  
+ test函数执行：
  
  ``` javascript
  console.log(a); //输出testEC.VE.DER.a
  a = function(){}; //改变a的值
  ```
  
+ test函数执行完毕，可执行上下文回退到GEC

4 执行b()

+ 创建b函数上下文：
  
  ``` javascript
  bEC = {
    VE: {
      DER: {
     	  //为空，没有变量
  	},
      outer: bEC.[[scope]] // = GEC
    }
  }
  ```
  
+ b函数声明绑定实例化，由于没有变量，跳过此步骤
  
+ b函数执行console.log(a); 查找 `bEC.VE.DER.a` 没有，然后找bEC.VE.outer , 就是GEC， GEC有变量a = 1; 所以输出 a = 1; 