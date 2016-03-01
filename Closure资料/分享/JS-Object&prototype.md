# 附录: JS object&prototype #
author: xiaofeng
2/28/2016 5:40:25 PM 

js是基于原型的对象,所有对象都是实例.
(*JS中对象是“键值对”的集合,并拥有一个连接到原型对象(prototype)的隐藏连接*)
(*对象字面量产生的对象连接到Object.prototype*)
(*Function.prototype连接到Object.prototype*)
(*函数对象连接到Function.prototype*)

(*\_\_proto\_\_内部原型, 构造器原型prototype*)
(*函数对象创建时附有一个prototype属性,prototype拥有一个constructor属性,值为该函数本身,prototype属性与连接到Function.prototype不同*)
(*注意: 不要混淆 prototype chain 与 scope chain*)

~~~
解释:
Object		->		function Object()
Function	->		function Function()
Function.prototype -> empty function
~~~

~~~
example:
var o1 = new Object()
var func = new Function()
var Foo = function() {}
var foo = new Foo()
~~~


1. FF实现了一个特定的属性\_\_proto\_\_, 指向创建函数的原型对象(prototype object)

	\_\_proto\_\_: 是原型对象的一个特定环境下的实现
	创建函数: 被用来创建该类型任意实例的函数, constructor


1. 所有的实例继承自创建他们函数(constructor)的原型对象(prototype object)

	实例 : return from new
	**继承 : XXX.\_\_proto\_\_**
	**原型对象 : XXX.prototype**

~~~
example:
o1.__proto__ === Object.prototype
func.__proto__ === Function.prototype
// function 是 对象
// Foo 是 Function的实例 Foo = new Function(code)
Foo.__proto__ === Function.prototype
// foo 是 Foo的实例
foo.__proto__ === Foo.prototype
// Function 是 function Function()
// function 是 对象
Function.__proto__ === Function.prototype
// Object 是 function Object()
// function 是 对象
Object.__proto__ === Function.prototype
~~~


??
// 不论原型对象的指针(\_\_proto\_\_)是否暴露, 所有的对象都会用原型对象指向创建自己的函数
Foo.__proto__ === Function.prototype

// 原型对象(prototype object)拥有一个属性constructor,指回拥有原型对象的函数
Foo.prototype.constructor === Foo

// Function -> function Function()
// Function 是 Function 自己的实例
Function.__proto__ === Function.prototype
Function.prototype.constructor === Function


// 原型只被用来

默认的原型对象可以被用户替换,这么做的时候,原型对象的constructor属性必须也手动修改,来修正js运行时使用默认原型对象添加的属性
~~~
function foo() { } ; var f1 = new foo();
f1.constructor === foo.prototype.constructor === foo  
//replace the default prototype object
foo.prototype = new Object();
//create another instance l
f1 = new foo();
//now we have:
f1.constructor === foo.prototype.constructor === Object
//so now we say:
foo.prototype.constructor = foo
//all is well again
f1.constructor === foo.prototype.constructor === foo
~~~


每个原型对象(XXX.prototype)自身默认是被Object() -> function Object()创建出来的,因此所有js中所有对象实例都从Object.prototype继承了属性
~~~
Function.prototype.__proto__ === Object.prototype
Array.prototype.__proto__ === Object.prototype
......
~~~

所有的对象查找自己未定义的属性时,自动上溯至原型链
~~~
function foo() { } 
f1 = new foo()
f2 = new foo()

foo.prototype.x = "hello" // 实例化之后 设置原型

f1.x  => "hello
f2.x  => "hello

f1.x = "goodbye"  // 设置同名属性, 原型链同名属性隐藏

f1.x  => "goodbye"  // 只将f1实例的x隐藏
f2.x  => "hello"
  
delete f1.x
f1.x  => "hello"  // 删除f1.x,重新查找到f1.prototype.x

foo.prototype.x = "goodbye" // 改变原型链属性直接改变所有实例
f1.x  => "goodbye"
f2.x  => "goodbye"
~~~


fixme !!!!!!!!!!!