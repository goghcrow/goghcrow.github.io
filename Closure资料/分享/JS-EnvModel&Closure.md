# JS的执行环境模型与闭包 #

author: xiaofeng
2/21/2016 10:10:51 PM 
2/22/2016 11:07:12 PM 
2/28/2016 5:39:11 PM
2/29/2016 9:29:10 AM  
3/2/2016 12:31:53 AM 

js的特性之 「对象」,「原型继承」
### **「闭包」**

**分享时长: 2~4h**

## 0. 闭包是什么鬼

摘自: https://zh.wikipedia.org/wiki/闭包_(计算机科学)

> 在计算机科学中「闭包」(Closure)又称「词法闭包」(Lexical Closure)或「函数闭包」(function closures),是引用了自由变量的函数。
> 
> 这个被引用的自由变量将和这个函数一同存在,即使已经离开了创造它的环境也不例外。所以,有另一种说法认为闭包是由函数和与其相关的引用环境组合而成的实体。
> 闭包在运行时可以有多个实例,不同的引用环境和相同的函数组合可以产生不同的实例。

> 闭包的概念出现于60年代,最早实现闭包的程序语言是Scheme。之后闭包被广泛使用于函数式编程语言如ML语言和LISP。很多命令式程序语言也开始支持闭包。

> 在一些语言中,在函数中可以「嵌套」定义另一个函数时,如果内部的函数引用了外部的函数的变量,则可能产生闭包。运行时,一旦外部的函数被执行,一个闭包就形成了,闭包中包含了内部函数的代码,以及所需外部函数中的变量的引用。其中所引用的变量称作上值(upvalue)。

~~~javascript
// my example
function outerFunc() {
	var outerVar = "hello"

	// Closure
	function innerFunc() {
		var innerVar = "world"

		// outerVar -> upvalue
		console.info("${outerVar} ${innerVar}")
	}

	innerFunc()
}

outerFunc()
~~~

> 闭包一词经常和匿名函数(Lambda)混淆。这可能是因为两者经常同时使用,但是它们是不同的概念。

摘自: ECMA-262
> A closure is a combination of a code block and data of a context in which this code block is created.
> 闭包是代码块和创建该代码块的上下文中数据的组合.

摘自: MDN Mozilla
> Closures are functions that 「refer」 to independent (free) variables.
> In other words, the function defined in the closure 'remembers' the environment in which it was created.

> 闭包是一种捕获了环境中自由变量的「引用」的函数.
> 换言之,使用闭包创建的函数可以「记住」它被创建时所处的环境.

摘自: 计算机科学
> A closure (also lexical closure or function closure) is a function or reference to a function together with a referencing environment ——「a table」 storing a reference to each of the non-local variables (also called free variables or upvalues) of that function.

# **WTF!!** #

缘起:《SICP》读到第三章, scheme语言引入set!之后,建立求值的环境模型,感觉这货用来解释**闭包**,秒杀之前看过一大坨资料。
这里分享下,基于环境模型来解释js中闭包,中间会穿插说到js的一些其他的特性。

(*javascript部分适合有点js基础的童鞋*)

## 1. JS环境模型

### 1. 环境

图例:
<pre>
+----------+
|          |
|          |
+----------+
方框表示框架,其中是一些 变量-值 约束
</pre>

- 一般的表达式都包含变量
  - 求值表达式的过程中用到变量的值
  - 为能找到变量的值,需要在某个地方记录它 
  - 记录变量约束值的结构称为**"环境(Environment)"**
- 环境确定了表达式求值的上下文
  - 没有环境,表达式求值就没有意义 
  - 即使求值 **1 + 1**,也需要环境为 **+** 提供意义

### 2. 环境模型

<pre>
            Env1
            +----------+
            |    frame1|
            | x = 3    |
            | y = 6    |
            +--+----+--+
               ^    ^
Env2           |    |  Env3
+-----------+  |    |  +------------+
|     frame2|  |    |  |      frame3|
| x = 6     +--+    +--+ y = 2      |
| z = 7     |          | z = 1      |
+-----------+          +------------+

        x 在 Env2 与 Env3中 的 值

Env1: [frame1]
Env2: [frame2 -> frame1]
Env3: [frame3 -> frame1]
</pre>

- **环境** 框架(frame)的链接结构
- **框架** 可空的 **表格(table)** ,项表示变量 **约束** ,在一个框架里每个变量至多有一个约束
- 每个框架有一个指向其外围框架的指针(**外围框架指针**),全局框架在上层,没有外围框架
- **一个变量在一个环境里的值**,就是它在该环境里的第一个有其约束的框架里的约束值

(*外围框架指针不要与this混淆,this为函数调用附加参数,与arguments相同*)

(*2/29/2016 9:05:01 PM  补充*)
(*每个函数都有一个scope与context与之关联,scope是基于函数的,context是基于对象的。
换而言之,scope是指函数被调用时的环境,同一函数每次被调用的的scope都是不同的,见下方环境模型函数调用规则。
而context在js中用this关键词表示,是当前执行代码的对象的“所属者”,用来描述当前代码如何被调用,见下文与js-function附录,函数的四种调用方式与this指向.
外围框架指针是环境模型的抽象概念,环境模型自身是对js Scope的抽象*)
~~~
table => 类比PHP实现,符号表
varibale object => 见下文EC,table可以理解为js函数scope chain内部的variable object
约束(Item) 	=> 每个varName(K)至多约束一个varValue(V)
变量在环境里的值lookup => 见 附录 kotlin代码
~~~

extra:
个人认为, 环境模型可以作为是ECMA规范中的作用域链的抽象描述

~~~
- Scope => 可访问的变量,对象,函数的集合
- Env => Scope chain
- 外围框架指针 => Scope Chain指针
~~~

js全局框架对应的作用域与宿主环境有关,浏览器:window,Nodejs:global


extra 1. :
用户创建的函数与build-in的函数都要保存在环境里,使用时通过检索环境(lookup)得到定义

extra 2. :
1. 假定js的全局框架(window),包含所有基本过程名意义的约束(这里将+=*\看成和内置过程alert同等地位的过程)
2. 没有执行代码前,系统的当前环境是全局环境
3. 随之程序进行,当前环境不断发生变化
	1. 可能在已有框架(包括全局框架)添加新约束
	2. 可能修改某个(某些)已有框架的约束
	3. 可能增加新的框架作为当前框架,原来的当前框架可能变新框架的外围框架

(*见 JS-EC*)

### 3. 规则

#### 1. 变量声明赋值与变量值修改
~~~javascript
var x
var y = 1
y = 2

var func = function() {
	y = 3
}

func()

console.info(undef) // undef is not defined 
undef = 1 // window.undef = 1
~~~

1. 变量声明: var <var-name> 
	- 在当前框架 建立 <var-name> 与 undefined的约束
2. 变量声明并赋值: var <var-name> = <value> 
	- 在当前框架 建立 <var-name> 与 value的约束
3. 变量值修改 <var-name> = <value>
	1. 在当前环境里查找 <var-name> 的约束。如果当前框架里有,约束就确定了,否则到外围框架去找。查找可以沿外围环境指针前进多步。
	2. 把找到的约束中变量的约束值修改为由 <value>表达式算出的值
	3. 如果环境中没有<var-name>的约束,查找到达全局框架但仍未找到,就抛出变量未定义的异常
		- ${var-name} is not defined 
		
**but:**
- 第3条,js中一旦修改未声明变量的值,会自动在window下建立该变量(全局变量)
- "use strict" 严格模式可以避免


#### 2. 创建函数

图例:
<pre>
+---+---+
| · | · |
+-+-+---+
上图序对用来表示[匿名]函数创建,左侧指向代码,右侧指向创建时环境
</pre>

~~~
// pair 序对
class Pair<T1, T2> {
	public T1 first;
	public T2 second;

	Pair(T1 first, T2 second) {
		this.first = first;
		this.second = second;
	}

	@override
	public String toString() {
		return "(" + first + "," + second + ")";
	}
}
~~~
创建JS函数得到一个**函数对象**,**函数对象**可以抽象成一个Pair(code, env)
- code指向函数的代码,包括参数和函数体
- env是环境指针,指向函数创建的环境,或者称为函数的上下文

(*以下example均以匿名函数方式创建函数, 见附录 js-function*)


~~~ javascript
var square = function(x) {
	return x * x
}
~~~

<pre>
            +-------------------------------+
global      |  其 他 变 量                   |
window +--> |                               |
            |  square:+--+                  |
            +-------------------------------+
                         |       ^
                         |       |
                         |       |
                         v       |
                     +-------+   |
                     | · | · +---+
                     +-+-+---+
                       |
                       v
                      (参 数 : x )
                      (函 数 体 : x * x)
</pre>

- 执行效果: 在全局环境新增square的约束,约束于创建的匿名函数
- 图解: 在原有其他变量约束之外新建立了square的约束
	- square 约束于一个匿名函数对象
	- 代码部分包括参数和函数体
	- 环境指针指向全局环境,也就是函数创建时所在的环境



#### 3. 函数调用

1. 建立新环境
	- 用函数对象的形参和实参建立一个新框架
	- JS会让每个函数接受两个附加参数,额外在新框架建立两个约束: 
		- this this指针,取决于调用模式(4中:方法|函数|构造器|apply/call[/bind])
		- arguments 参数列表(类数组对象)
	- 函数体内的变量声明或函数创建(声明前置)会在当前框架内建立新约束
	- 建立以新框架为当前框架,以环境指针为外围环境的新环境
2. 在新环境里调用函数(以下均指函数调用方式)

example:
~~~ javascript
square(5)
~~~

<pre>
            +-------------------------------+
global      |  其 他 变 量                   |
window +--> |                               |
            |  square:+--+                  |
            +--------------------+-------+--+
                         |       ^       ^
                         |       |       |     环境E1+----+
                         |       |       |                |
                         ^       |       |                | E1当前框架
                     +-------+   |  +----+----------------v-----+
                     | · | · +---+  | x = 5                     |
                     +-+-+---+      | this = window             |
                       |            | arguments = {0:5,length:1}|
                       v            +---------------------------+
                      (参 数 : x )
                      (函 数 体 : x * x)
</pre>

图解:

1. 先创建新环境E1,建立一个新框架作为当前框架,其中形参x约束到实参5
2. 附件this,arguments两个变量,建立约束
3. 在E1求值函数体x * x 得到结果25

#### 规则总结:

- 规则中的外围环境通过环境指针确定,未必指向全局框架
- 创建函数对象时确定了环境,建立新环境的规则都一样

##### 1. 创建函数基本规则: 

1. 在环境 E 里创建一个函数(lambda表达式)对象
2. 建立一个函数对象 其代码是该 函数体 和 参数
3. 其环境指针指向 E
4. [赋值: 将函数对象约束到环境E]

##### 2. 调用函数的基本规则:(将一个函数对象应用于一组实参的过程)

1. 构造一个新框架,其中存入函数的形参(附件this与arguments)与对应实参的约束
2. 添加变量与函数声明作为框架约束(声明前置)
3. 该框架以函数对象的环境指针指向的框架作为外围框架,形成一个新环境
2. 在这个新环境中求值函数体


### 4. 简单函数的定义与调用

~~~ javascript
var square = function(x) {
	return x * x
}

var sumOfSquare = function(x, y) {
	return square(x) + square(y)
}

var f = function(a) {
	return sumOfSquare(a + 1, a * 2)
}

f(5)
~~~

sample:
求值f(5)

<pre>
                  +-----------------------------------------------------+
                  |                                                     |
global            |  square:+------------------------------------+      |
                  |                                              |      |
window +--------> |  sumOfSquare:+------------+                  |      |
                  |                           |                  |      |
                  |  f:+-+                    |                  |      |
                  +------------------+----------------+----------------++
                         |           ^        |       ^          |     ^
                         |           |        |       |          |     |
                         |           |        |       |          |     |
                         |           |        |       |          |     |
                         |           |        |       |          |     |
                         v           |        v       |          v     |
                     +---+---+       |    +---+---+   |      +---+---+ |
                  +--+ · | · +-------+  +-+ · | · +---+   +--+ · | · +-+
                  |  +---+---+          | +---+---+       |  +---+---+
                  |                     |                 |
                  |                     |                 |
                  v                     v                 v
                parameter:a            parameter: x, y   parameter: x
                body: sumOfSquare(     body: square(x)   body: x * x
                        a + 1,               +
                        a * 2)               square(y)



                      +-----------------------------------------------------+
                      |                                                     |
    global            |                                                     |
                      |                                                     |
    window +--------> |                                                     |
                      |                                                     |
                      |                                                     |
                      +----+--------------+-------------+--------------+----+
                           ^              ^             ^              ^
                           |              |             |              |
                    +E1    |      +E2     |      E3     |      E4      |
                    |      |      |       |      +      |      +       |
    	    		| +---+----+  |  +----+----+ |  +---+---+  |  +----+---+
					+>| a: 5   |  +> | x: 6    | +> | x: 6  |  +> | x: 10  |
                      |        |     | y: 10   |    |       |     |        |
                      +--------+     +---------+    +-------+     +--------+

           sumOfSquare(a+1, a*2) square(x) + square(y)  x * x       x * x

注意: E1 ~ E4环境隐藏this与arguments约束

</pre>

- 每次函数调用建立新框架,同一函数的不同调用各自建立框架,相互无关
- 不关注返回值与局部状态的对象计算情况

### 5. 框架与局部状态

~~~javascript
var makeWithdraw = function(balance) {
	return function(amount) {
		if(balance >= amount) {
			balance -= amount
			return balance
		} else {
			console.error("Insufficient funds")
		}
	}
}

// 提款1
var w1 = makeWithdraw(100)
w1(50)
w1(50)

// 提款2
var w2 = makeWithdraw(100)
w2(50)

~~~

<pre>
var makeWithdraw = function() {...}

global    +----------------------+
window    |                      |
+         |makeWithdraw: +--+    |    
|         |                 |    |
+-------> +-----------------+--+-+
                            |  ^  
                            |  |   
                         	v  |   
                       +----+--+-+
                  +----+ ·  |  · +
                  |    +----+----+
                  v
                  parameter: balance
                  body: ...

</pre>

<pre>
var w1 = makeWithdraw(100)

         +---------------------------------------------------+
global   |  makeWithdraw:+-------------------------+         |
window   |                                         |         |
+        |  w1:+                                   |         |
+------> +------------------------^----------------+-------+-+
               |                  |                v       ^
               |           +------------+      +---+---+   |
               |   E1 +--->+balance: 100|      | · | · +---+
               |           +----+-------+      +-+-+---+
               |                ^                |
               v                |                v
           +-------+            |              parameter: balance
           | · | · +------------+              body: ......
           +-+-+---+
             |
             v
           parameter: amount
           body:
           ... balance -= amount ...

</pre>

<pre>
w1(50)

         +---------------------------------------------------+
global   |  makeWithdraw: ...                                |
window   |                                                   |
+        |  w1:+                                             |
+------> +-----+------------------^--------------------------+
               |                  |
               |           +------+-----+
               |   E1 +--->+balance: 100|
               |           +----+-----+-+
               |                ^     ^
               v                |     |
           +-------+            |     +------+
           | · | · +------------+        +---+-------+
           +-+-+---+                     |amount: 50 |
             |                           +-----------+
             v                           balance -= amount
           parameter: amount
           body: ...
           ... balance -= amount ...

         +---------------------------------------------------+
global   |  makeWithdraw: ...                                |
window   |                                                   |
+        |  w1:+                                             |
+------> +-----+------------------^--------------------------+
               |                  |
               |           +------+-----+
               |   E1 +--->+balance: 50 |
               |           +----+-------+
               |                ^
               v                |
           +-------+            |
           | · | · +------------+
           +-+-+---+
             |
             v
           parameter: amount
           body: ...
           ... balance -= amount ...

</pre>

<pre>
var w2 = makeWithdraw(100)
w2(50)

         +---------------------------------------------------+
global   |  makeWithdraw: ...                                |
window   |  w2:+--------------------------+                  |
+        |  w1:+                          |                  |
+------> +-----+------------------^-------+-------------^----+
               |                  |                     |
               |           +------+-----+    +----------+---+
               |   E1 +--->+balance: 50 |    |balance: 100  +<----+ E2
               |           +----+-------+    +--------+-----+
               |                ^                     ^
               v                |                     |
           +-------+            |     +---+---+       |
           | · | · +------------+     | · | · +-------+
           +-+-+---+                  +-+-+---+
             |       +------------------+
             v       v
           parameter: amount
           body: ...



</pre>

图解:
1. var makeWithdraw = func(){} : 全局环境约束makeWithdraw
2. makeWithDraw(100) : 函数调用, 建立环境E1, 外围框架指向全局
3. 在E1求值匿名函数,建立一个函数对象,其环境指针指向求值环境E1
4. var w1 =: 赋值,建立全局环境约束
5. w1(50): 函数调用,建立新框架,环境指针指向w1的环境E1,从环境框架里找变量的值
6. balance -= amount : 修改E1环境中balance值为50
7. w1(50): 再次调用w1,建立建立一个新框架,但与上面建立的框架无关,其外围框架仍是E1
8. 对函数体求值将再次找到包含balance的框架E1,并修改balance的值
9. 重复2、3、4,建立环境E2, 建立w2约束

- w1的局部状态与w2的局部状态无关,各自独立变化
- w1与w2的代码部分完全相同
	- 共享一份代码还是各有一份,是系统实现的细节
	- 不同实现方式不影响程序语义,但影响资源消耗,聪明的编译器共享代码,节省内存消耗

### 6. 内部定义(内部函数)
~~~ javascript
var square = function(x) {
	return x * x
}
var average = function(a, b) {
	return (a + b) / 2
}
var sqrt = function(x) {
	var goodEnough = function(guess) {
		return Math.abs(square(guess) - x) < 0.001
	}
	var improveGuess = function(guess) {
		return average(guess, x / guess)
	}
	var sqrtIter = function(guess) {
		if(goodEnough(guess)) {
			return guess
		} else {
			return sqrtIter(improveGuess(guess))		
		}
	}
	return sqrtIter(1.0)
}

sqrt(2)
~~~

<pre>
global     +---------------------------------------------------+
window     | sqrt:                                             |
+          |                                                   |
+--------> +-----+--------------------------------+------------+
                 ^                                ^
                 |     call sqrt(2)               |
      +---+---+  |    +---------------------------+------------+
 +----+ · | · +--+    | x: 2                                   |
 |    +---+---+       | goodEnough:+------------------------+  |
 v              E1 +--+ improveGuess:|------------+         |  |
 parameter: x         | sqrtIter:+-------+        |         |  |<--+
 body: ...            +-+---+------------+----+---+----+----+--+   |
                        ^   ^            |    ^   |    ^    |      |
                        |   |            v    |   |    |    v      |
               +--------++  |        +---+---+|   |    |+---+---+  |
        E2 <---+guess: 1 |  |        | · | · |+   |    || · | · +--+
               +---------+  |        +-+-+---+    |    |+-+-+---+
         call sqrtIter(1)   |          |          v    |  |
                       +----+---+      |      +---+---+|  |
                E3 <---+guess: 1|      |      | · | · ++  |
                       +--------+      |      +-+-+---+   |
                call goodEnough(1)     |        |         |
                                       v        v         v
                                    para:guess para:guess para:guess
                                    body:...   body:...   body:...

</pre>

图解:
1. sqrt(2),建立框架E1,其中有形参约束,与内部函数约束
2. sqrtIter(1),建立框架E2
3. goodEnough(1),建立框架E3
4. improveGuess(1),建立框架E4,E4外部框架指向E1


- 在建立函数对象时
	- (**private**)内部函数的名字与相应匿名函数对象的约束在一个局部框架里,与其他框架里的同名对象,变量或过程无关
	- (**upvalue**)内部函数对象的环境指针指向外部函数调用时的环境(E1:sqrt调用产生的环境),因此内部函数可以直接使用其外部函数的局部变量,形式参数等
- 每次调用有内部函数的外部函数时,将新建一个框架
	- 包括重新建立其中的各内部函数对象
	- 代码的处理见前面说明,不同函数对象之间是否共享代码是系统的实现细节,不影响语义


### 2. 闭包

<pre>
                         ^
                         |
                         |
                +------------+
       Env  --->+balance: 100|
     +          +----+-------+
funcObj              ^
                     |
+---+---+            |
| · | · +------------+
+-+-+---+
  |
  v
parameter: amount
body: balance -= amount

</pre>

### 3. 闭包的例子与应用
#### 1. 闭包中的自由变量是ref不是copy,另一个例子见备注"一个陷阱"
~~~
var printX = (function() {
	var x = 1
	return x++, function() {
		console.info(x)
	}
}())

printX() // 2
~~~

#### 2. 同一函数的每个闭包都是一个新的实例
~~~
var newStack = function() {
	var container = []
	return {
		push: function() {
			// va
			[].slice.apply(arguments).forEach(function(el) {
				container.push(el)
			})
		},
		pop: function() {
			return container.pop()
		},
		print: function() {
			console.info("[" + container.join(", ") + "]")
		}
	}
}

var stackA = newStack()
stackA.push(1)
stackA.push(2, 3)
stackA.pop()
stackA.print() // [1, 2]

var stackB = newStack()
stackB.push('a', 'b', 'c')
stackB.print() // [a, b, c]
~~~
用环境模型图理解


#### 3. 封装
~~~
var newPerson = function(name) {
	// private prop
	var _name = name
	
	// private method
	var _method = function() {
		console.info("hello, " + _name)
	}
	
	// public
	return {
		getName: function() {
			return _name
		},
		setName: function(name) {
			_name = name
		},
		hello: function() {
			_method()
		}
	}
}

var p1 = newPerson("xiaofeng")
console.info(p1.getName())
p1.hello()
~~~

#### 4. 单例
~~~
var counter = function() {
	var i = 0
	return {
		reset: function() {
			console.info(i = 0)
		},
		incr: function() {
			console.info(++i)
		}
	}
} ()

counter.incr()
counter.incr()
counter.reset()
~~~
counter通过一个IIFE返回一个单例对象,隐藏了计数器内部变量i
(*IIFE:Immediately Invoked Function Expression*)


#### 5. 类库惯例(AMD,CMD,ES6之前)
~~~
(function(win, undefined){
	// win.youNamespace = todo
})(window)
~~~

#### 6. 事件句柄,回调函数
~~~
// 3/2/2016 12:54:32 AM 
// 一个有意思的演示代码
// http://image.baidu.com/

var analyze = {}
$("body").on("click", "img", function(e) {
	var name = e.target.src.split("/").pop() || "unknown"
	analyze[name] = (analyze[name] || 0) + 1
	console.info("clicked analyze:", analyze)
	return false // e.preventDefault()
})

// 或者更进一步
$("body").on("click", "img", (function() {
	var analyze = {}
	return function(e) {
		var name = e.target.src.split("/").pop() || "unknown"
		analyze[name] = (analyze[name] || 0) + 1
		console.info("clicked analyze:", analyze)
		return false	
	}
}()))
~~~

#### 7. curry: 闭包与多参数函数
数学角度看闭包
~~~
// lambda函数只接受一个参数,如何表示多参数函数,于是引进curry
var add = function(a, b, c) {
	return a + b + c
}
// 实际上是:
var add = function(a) {
	return function(b) {
		return function(c) {
			return a + b + c
		}
	}
}

var funcAdd1 = add(1)
var funcAdd12 = funcAdd1(2)
var result = funcAdd12(3)
console.info(result)
~~~

#### 8. lambda lifting: 减少函数嵌套与闭包编译器优化
编译器角度看闭包
~~~
var add = function(a) {
	return function(b) {
		return function(c) {
			return a + b + c
		}
	}
}
~~~

~~~
var add = function(a) {
	// 闭包依赖自由变量 a
	var addA = function(b) {
		// 闭包依赖自由变量 a, b
		var addB = function(c) {
			return a + b + c
		}
		return addB 
	}
	return addA
}

add(1)(2)(3)
~~~

~~~
var add = function(a) {
	// 取消对自由变量a依赖
	var addA = function(b, a_1) {
		// 闭包依赖自由变量 a, b
		var addB = function(c) {
			return a_1 + b + c
		}
		return addB 
	}
	return addA
}
// 变量a,调用方多传参数
add()(2, 1)(3)
~~~

~~~
// 不需要对自由变量依赖,优化掉闭包
var addA_1 = function(b, a_1) {
	// 闭包依赖自由变量 a_1, b
	var addB = function(c) {
		return a_1 + b + c
	}
	return addB 
}

var add = function() {
	return addA_1
}
add()(2, 1)(3)
~~~

~~~
// 取消对自由变量b, a_1依赖
var addA_1 = function(b, a_1) {
	// 闭包依赖自由变量 a_1, b
	var addB = function(c, b_1, a_2) {
		return a_2 + b_1 + c
	}
	return addB 
}

var add = function() {
	return addA_1
}
add()()(3, 2, 1)
~~~

~~~
// 优化掉闭包
var addB_1 = function(c, b_1, a_2) {
	return a_2 + b_1 + c
}


var addA_1 = function() {
	return addB_1
}

var add = function() {
	return addA_1
}
add()()(3, 2, 1)
~~~


~~~
// inline 最终版本
var add = function(c, b_1, a_2) {
	return a_2 + b_1 + c
}
add(3, 2, 1)
~~~
lambda lifting远比以上复杂

9. 见附录 tms-basic项目代码实例

### 4. 程序执行环境
#### 1. 执行环境(Execution Context)
ECMA262:
All the ECMAScript program runtime is presented as the execution context (EC) stack, where top of this stack is an active context.

1. 运行时 -> EC栈
2. 栈顶 -> active context

<pre> 
    EC STACK
+--------------+
|--Active  EC--| <----- TOP
+--------------+
|   ......     |
+--------------+
|   ......     |
+--------------+
|   ......     |
+--------------+
|   Global EC  |
+--------------+
</pre>

js运行在一个逻辑上的stack结构中,js解释器初始化,默认进入global EC,每次函数调用创建一个新的EC,每创建一个EC都将push到ECstack的顶部,函数return,当前EC从stack pop出去,控制权转交下方EC.
<pre>
EC构成:
+--------------------------------------------------+
|Excution Context                                  |
|                                                  |
+--------------------------------------------------+
|Variable Object    |{vars, function declarations, |
|(Activation Object)|arguments,...}                |
+--------------------------------------------------+
|Scope Chain        |Varibale Object,              |
|                   |all parent Scope Chain        |
+--------------------------------------------------+
|this Object        |Context Option                |
+----------------+---------------------------------+
</pre>

每个EC都可以被分为创建阶段(creation phase)与执行阶段(excution phase),创建阶段,
1. 脚本解释器会首先创建一个Variable Object,包括所有变量与函数的声明与arguments,
2. 然后scope chain初始化,
3. 确认this的值,
4. 最后是执行阶段,代码被解释与执行

#### 2. 作用域链(Scope Chain)
Scope chain内部是ecStack中每个ec的VO.被用来决定变量的访问与标识符解析.
(*环境模型的ec表示*)
~~~javascript
// this === window
first()
function first() {
	second()
	function second() {
		third()
		function third() {
			forth()
			function forth() {
				// ...
			}
		}
	}
}
~~~
<pre> 
    EC STACK
+--------------+
|   forth      | <----- TOP
+--------------+
|   third      |
+--------------+
|   second     |
+--------------+
|   first      |
+--------------+
|   Global EC  |
+--------------+

global       +---------------------------------------------------------+
window       |                                                         |
+            | first:+                                                 |
+----------> |  +----+                                                 |
             +--|-----+--------+---------------------------------------+
                |     ^        ^  
                |     |        |  
                v     |  +-------+
             +--+--+  |+-+:second| <- first invoke env
             | ·|· +--+| |       |
             +--+--+   | +--+--+-+
              First    |    ^  ^     +-------+
                       v    |  |     |:third | <- second invoke env 
                    +--+--+ |  +-----+       |
                    |· |· |-+        +-----+-+             
                    +--+--+                ^      +-------+
                    Second                 |      |:forth | <- third invoke env
                                           +------+       |
                                                  +----+--+   +-------+
                                                       ^      |       |
                                                       +------+       |
                                                              +-------+

</pre>

同名变量规则同上,本地优先级高于parent scope;简而言之,在函数的ec中每次试图访问一个变量时,look-up过程总是从函数自身的VO开始.如果标识符(变量名)在VO中木有找到,将会爬升并检测ScopeChain中的每一个ec,直到找到匹配的变量名或者检测到global-ec.

#### 3. 闭包(Closure)
......


总结:
**js引擎会为每个函数体提供三个对属性,链式作用域属性使得闭包成为可能,让函数记住它被创建时所处环境的,环境模型是个人对Ecma规范中VO与ScopeChain的抽象**.


### 5. (选)js闭包两个eval hacker

#### 1. 突破闭包作用域

~~~
var x = 'outer';

(function(){
    var x = 'inner'

	// Direct call
	// 左值引用, activeEC

    eval('console.log(x)');   //inner

	// Indirect call
	// 右值引用, Global EC

	(0,eval)('console.log(x)')//outer

})()
~~~

ECMA规范指出,eval有两种调用模式:Direct call & Indirect call,两种调用模式会初始化不同的EC,	逗号操作符用于赋值的干预下,eval变为右值引用,间接调用,初始化globalEC作为activeEC

(0, eval)
("", eval)
(null, eval)
......

ECMA-262:

> If there is no calling context or if the eval code is not being evaluated by a direct call (15.1.2.1.1) to the eval function 
> then initialize the execution context as if it was a global execution context using the eval code as described like this below
> Set the VariableEnvironment to the Global Environment.
> Set the LexicalEnvironment to the Global Environment.
> Set the ThisBinding to the global object.

画环境模型的图来解释,就是eval创建的函数环境指针指向了global env
~~~
// 作用: 一种突破scope chain引用全局元素的方式
(function() {
	var window = {x: 1, y: 2}
	var nativeWindow = this || (0, eval)("this");
	console.info(nativeWindow)
}())
~~~

#### 2. 闭包外直接访问闭包内部自由变量与局部变量

~~~
(function(window) {
	var $parent = {
		name: "xiaofeng"
	}

	window.$$ = {
		magic: function(code) {
			eval("(" + code + "())")
		}
	}
} (window))


var $parent = {
	name: "lvmeng"
}

console.info($parent.name)

$$.magic(function() {
	console.info($parent.name)
})
~~~
左值引用的eval进行直接调用,ec初始化自eval当前ec,或者说eval内创建函数的环境指针指向eval执行的环境,从而使得该函数调用时生成的框架的外围框架指针指向eval执行的环境,总之仍旧满足环境模型的规则.**but**,eval的执行效率在不同的引擎上得不到保证,慎用


### 附录:

#### 1. C(无block扩展)、Java(8以前)没有传统意义的闭包
- 从函数调用堆栈角度讲,闭包就是就是函数的“堆栈”在函数返回后并不释放,所以也可以理解为这些函数堆栈并不在栈上分配而是在堆上分配
- php函数调用也是基于stack实现,php中对闭包的支持是通过use关键词,显示引用内部函数作用域外变量与参数 
~~~php
<?php
function createSayHello($who) {
	$hello = "Hello";
	return function($to) use($who, $hello) {
		return "$who say $hello to $to\n";
	}
}

$xiaomingSayHelloTo = createSayHello("小明");
echo $xiaomingSayHelloTo("小红"); // 小明 say hello to 小红
echo $xiaomingSayHelloTo("小丽"); // 小明 say hello to 小丽

var_dump($xiaomingSayHelloTo instanceof Closure); // bool(true)
// php中匿名函数用一个叫做Closure的类来实现
// Closure意为闭包,个人认为有误,闭包事实是用use来手动实现的
~~~

~~~ C
// 栈上变量在函数返回后清空
// error
int* func() {
	int val = 1;
	return &val;
}
~~~

Lambdas: local variables need final, instance variables don't
http://stackoverflow.com/questions/25055392/lambdas-local-variables-need-final-instance-variables-dont

为什么java8 lambdas的参数需要final

#### 2. 闭包的一个“坑”
~~~
for(var i=0; i<5; i++) {
	setTimeout(function() { console.info(i)}, 0)
}
// output: 5 5 5 5 5

for(var i=0; i<5; i++) {
	(function(i) {
		setTimeout(function() { console.info(i)})
	} (i))
}
// output: 0 1 2 3 4
~~~
实际上正好说明,闭包对自由变量是一种引用

#### 3. Environment & Scope Chain 示例代码

**kotlin**
- playground: http://try.kotlinlang.org/
~~~
// @2015-02-20
import java.util.*

class Env(val parent: Env? = null) {

	// Pair(var-type, var-value)
    val table = LinkedHashMap<String, Pair<String?, Any?>>()

    fun put(name: String, type: String?, value: Any?) {
        table[name] = type to value
    }

    fun lookupLocal(name: String): Pair<String?, Any?>? = table[name]

    fun lookup(name: String): Pair<String?, Any?>? {
        val pair = lookupLocal(name)
        if(pair != null) {
            return pair
        } else if(parent != null) {
            return parent.lookup(name)
        } else {
            return null
        }
    }

    override fun toString(): String {
        val sb = StringBuffer()
        sb.append("{\n")
        table.map { "${it.key}: ${it.value.second}" }.joinTo(sb, ",\n", "", ",\n")
        sb.append("parent: ${parent}")
        sb.append("\n}")
        return sb.toString()
    }
}

fun main(args: Array<String>) {
    val env1 = Env()
    env1.put("name", "string", "xiaofeng")
    env1.put("age", "int", 25)

    val env2 = Env(env1)
    println(env1)
    println(env2)
    
	println(env2.lookupLocal("name"))
    println(env2.lookup("name"))

}
~~~

#### 4. tms-basic 项目代码实例

~~~
    /**
     * 生成一个按delay防重的函数
     * @param func
     * @param delay 多长时间内只执行一次
     * @returns {Function}
     */
    var preventRepeat = function(func, delay) {
        var timeout
        delay = parseInt(delay, 10) || 100
        // func = (typeof func === "function") || function() {}
        return function() {
            if(timeout) {
                clearTimeout(timeout)
            }
            timeout = setTimeout(func, delay)
        }
    }


	$('body').click(preventRepeat(function(){
		console.info("clicked")
	}, 500))
~~~

~~~
    /**
     * curry
     *
     * @param  func function 需柯里化函数
     * @param  variableArg  boolean 是否可变参数 默认false
     * @param  autoCompleted  boolean 参数补全之后是否自动求解 默认true
     * @return mixed
     * @author xiaofeng
     */
    var curry = function(func, variableArg, autoCompleted) {
        variableArg = !!variableArg
        autoCompleted = autoCompleted === void 0 ? true : !!autoCompleted
        var completed = false

        var helperFn = function(argsCarry, argsLeft) {
            var retFn = function() {
                var args = [].slice.apply(arguments)
                argsLeft = argsLeft - args.length
                completed = (!variableArg && argsLeft <= 0)
                if(completed && !!autoCompleted) {
                    return func.apply(undefined, argsCarry.concat(args))
                } else {
                    return helperFn(argsCarry.concat(args), argsLeft)
                }
            }
            retFn.valueOf = function() {
                return (completed || variableArg) ? func.apply(undefined, argsCarry) : void 0
            }
            return retFn
        }

        return helperFn([], func.length)
    }

    var add = function(a, b, c) {
    	return a + b + c
    }
    curry(add)(1)(2)(3) == 6
    curry(add)(1)(2, 3) == 6
    var sum = function() {
        return [].slice.apply(arguments).reduce(function(a, carry) { return a + carry}, 0)
    }
    curry(sum, true)(1) == 1
    curry(sum, true)(1, 2) == 3
    curry(sum, true)(1)(2) == 3
    curry(sum, true)(1)(2)(3, 4) == 10
~~~

~~~
    /**
     * memorized
     * 缓存方法辅助函数
     *      默认缓存key生成函数 [].slice.apply(arguments).join("&")
     *      可通过第二个参数自定义缓存key,key生成函数接收原函数调用参数作为自己的参数
     *
     * @param  func function 需要缓存结果的方法
     * @param  createKeyFunc  function cachedKey使用调用参数生成 默认参数toString & 连接
     * @return mixed
     * @author xiaofeng
     */
    var memorized = function(func, createKeyFunc) {
        if(typeof func !== "function") {
            throw new Error("argument error")
        }
        if(typeof createKeyFunc !== "function") {
            createKeyFunc = function() {
                // 默认缓存key生成方法
                return [].slice.apply(arguments).join("&")
            }
        }

        var cache = {}

        return function() {
            // apply ctx = this
            // 使用者可自行调用bind(ctx) 绑定上下文
            var result
            var args = [].slice.apply(arguments)
            var key = createKeyFunc.apply(this, args)
            console.info("cached key:", key)
            if(cache[key] === void 0) {
                result = func.apply(this, args)
                cache[key] = result
                console.info("cached key miss:", {key: key, result: result})
            } else {
                result = cache[key]
                console.info("cached key hit:", {key: key, result: result})
            }
            return result
        }
    }

	var add = memorized(function(x, y) {
		return x + y
	})


	// 网上的一段代码,用bind实现原生的curry
	// 推荐这种使用方式
	function converter(toUnit, factor, offset, input) {
	    offset = offset || 0;
	    return [((offset+input)*factor).toFixed(2), toUnit].join(" ");
	}
 
	var milesToKm = converter.bind(undefined, 'km', 1.60936, 0);
	var poundsToKg = converter.bind(undefined, 'kg', 0.45460, 0);
	var farenheitToCelsius = converter.bind(undefined, 'degrees C',0.5556, -32);
 
	milesToKm(10);            // returns "16.09 km"
	poundsToKg(2.5);          // returns "1.14 kg"
	farenheitToCelsius(98);   // returns "36.67 degrees C"
~~~

### 参考:
1. 环境模型: 《SICP》
2. 环境模型: 《SICP》译者 裘宗燕老师课件
3. js基础知识: 《javascript精粹》
4. 闭包概念: https://zh.wikipedia.org/wiki/闭包_(计算机科学)
5. 闭包概念与ECMA规范原文: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Language_Resources
8. ECMA规范译文: http://yanhaijing.com/es5/#143
10. EC: http://goddyzhao.tumblr.com/post/10020230352/execution-context
6. js原型对象布局: http://www.mollypages.org/tutorials/js.mp
7. http://zora.ghost.io/tan-tan-bi-bao-yu-lambda/
12. http://zora.ghost.io/tan-tan-0evalyu-eval/
13. http://typeof.net/2014/m/introducing-lambda-lifting.html
9. http://ryanmorr.com/understanding-scope-and-context-in-javascript/
13. ascii 制作: http://asciiflow.com/
14. MD编写: MarkdownPad2