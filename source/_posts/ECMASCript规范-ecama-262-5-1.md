layout: title
title: "ECMASCript规范(ecama-262-5.1)"
date: 2015-04-13 16:49:10
tags: 规范
---
  
这可能是个天坑系列，规范神马的真是看着让人崩溃。随时有可能放弃。

>##4.3.2 primitive value
>原始值：原始值包括 Undefined, Null, Boolean, Number, String
>表示语言的最底层实现
>##4.3.7 object
>原生对象包括内置对象和自定义对象
>宿主对象指宿主环境(浏览器)实现的对象

<!--more-->

>##5 词法和文法的解释 6 源代码文本 7 词法
>**没看懂！！**
>## 7.9.1 分号插入机制
>+ 从左到右解析，如果遇到不符合语法的token(词法)，可以理解为和接下来的语句连起来执行的时候：
>  - 下一个token和之前的代码之间有换行，在下一个token前加";"
>  - 下一个token是"}" ,在它前面加 ";"
>+ 程序结束的位置加";"
>+ 受限产生式前面如果有换行，加分号。
>  - 受限表达式：PostfixExpression :
			LeftHandSideExpression [no LineTerminator here] ++
			LeftHandSideExpression [no LineTerminator here] --
			ContinueStatement :
			continue [no LineTerminator here] Identifier ;
			BreakStatement :
			break [no LineTerminator here] Identifier ;
			ReturnStatement :
			return [no LineTerminator here] Expression ;
			ThrowStatement :
			throw [no LineTerminator here] Expression ;
>+ 自动分号插入从来不会插入成 for 语句头部的两个分号之一。
>##8.0 类型
>ECMAScript language types(ecmascript语言类型): Undefined, Null, Boolean, String, Number, and Object.（其中null现在规范是object其实是个bug，但他的确是一种类型）
>specification types(规范定义类型，用于描述ECMAScript语法和ECMAScript类型):Reference, List, Completion, Property Descriptor, Property Identifier, Lexical Environment, and Environment Record. 
>###8.6.1
>##10.0
>###10.1 可执行代码
>Ecmascript中有3种可执行代码：
>
>+ 全局代码
>+ eval代码
>+ 函数代码
###10.2 Lexical Environments （词法环境）
> _词法环境_是一个规范类型，用来描述特定变量和标识符的关系以及基于词法嵌套结构的ECMASCRIPT代码。词法环境由一个_环境标示_和一个可以为空的外部_词法环境_组成。一般来说，一个_词法环境_和一些指定的ECMASCRIPT代码结构有关，比如 _FunctionDeclaration_(函数声明) ，_WithStatement_(with语句)，或者是_Catch_ 内容，当上述代码_evaluated_(评估) 的时候，会创建一个新的**词法环境**。

> 一个_Environment Record_(环境标示) 在它关联的词法环境中创建，记录着标识符绑定。

> 外部环境引用用来模拟_词法环境_的逻辑嵌套。一个内部词法环境的的外部环境引用就是这个内部词法环境的逻辑包裹的环境的词法环境。一个外部的词法环境当然也拥有它自己的词法环境。一个外部词法环境可以提供给多个它的内部环境。比如，如果一个 _FunctionDeclaration_(函数声明) 包含两个内部的_FunctionDeclaration_(函数声明)那么这两个函数声明就有同一个外部词法环境--它们共同包裹函数的词法环境。

> 词法环境和环境标示的值是严格符合规范的，不应该符合任何加工过的规范。一个ECAMAscript的程序不能直接访问或者操作这些值。

###10.2.1 Environment Records(环境标示)
> 环境标示的值可能是下面两种：_declarative environment records_ (声明式环境标示)和_object environment records_ (对象式环境标示).声明式环境标示用于描述类似于下面这些标识符绑定的效果：函数声明、变量声明和catch内部语句等直接关联ECMAScript语言值的标识符绑定。对象式环境标识用于描述ECMAScript元素的效果比如 全局程序或者with语句 这些关联特定对象的标识符绑定。

> 出于规范的目的，环境标示可以看作是一个简单的面向对象的层次结构，它有两个具体的子类，声明式环境标示和对象式环境标示。这些抽象的类包括下面这些方法

具体方法等使用的时候再翻译


###10.2.1.1 声明式环境绑定
> 每一个声明式环境标示都和一个包含变量和(或)函数声明的ECMAScript程序作用域有关系。声明式环境标示通过作用域内的声明绑定了一系列标识符。

> 除了所有环境记录项都支持的可变的绑定，声明式环境标示还提供了不变绑定。一个不变的绑定就是指标识符和值不会绑定之后不会发生改变。创建和初始化不可变绑定是两个独立的过程，因此类似的绑定可以处在已初始化阶段或者未初始化阶段。

### 10.2.1.2 对象式环境标示

> 每一个对象式环境记录项都有一个关联的对象，这个对象被称作 绑定对象 。对象式环境记录项直接将一系列标识符与其绑定对象的属性名称建立一一对应关系。不符合IdentifierName 的属性名不会作为绑定的标识符使用。无论是对象自身的，还是继承的属性都会作为绑定，无论该属性的 [[Enumerable]] 特性的值是什么。由于对象的属性可以动态的增减，因此对象式环境记录项所绑定的标识符集合也会隐匿地变化，这是增减绑定对象的属性而产生的副作用。通过以上描述的副作用而建立的绑定，均被视为可变绑定，即使该绑定对应的属性的 Writable 特性的值为 false。对象式环境记录项没有不可变绑定。

> 对象式环境记录项可以通过配置的方式，将其绑定对象合为函数调用时的隐式 this 对象的值。这一功能用于规范 With 表达式（12.10 章 ）引入的绑定行为。该行为通过对象式环境记录项中布尔类型的 provideThis 值控制，默认情况下，provideThis 的值为 false。

> 环境记录项定义的方法的具体行为将由以下算法给予描述。





###词汇表
primitive value ： 原始值  
constructor ： 构造函数  
native object ： 原生对象  
built-in object ： 内置对象  
host object ： 宿主对象  
productions ： 产生式   
Environment Records : 环境标示  
Lexical Environments ： 词法环境   
declarative environment records ： 声明式环境标示   
object environment records ： 对象式环境标示     
