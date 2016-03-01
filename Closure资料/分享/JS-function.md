# 附录: JS function #
author: xiaofeng
2/28/2016 5:39:11 PM 

1. 函数是js中唯一拥有作用域的结构(无块级作用域),闭包的创建依赖于环境
(*ES6中,let关键词可以声明块级作用域的变量*)


2. 第一类对象,高阶函数(接受函数参数,返回函数)。
因为是对象,所以函数可以
	- 像任何其他对象一样使用
	- 可以赋值给变量 var func = function() {}
	- 可以存储在对象或者数组中 var arr = [function(){}, ......]
	- 可以当作函数参数,返回值 [].map(function(){ return })
	- 可以拥有方法 var func = function() {}; func.method = function() {}
3. 四种调用方式与this指向......
......

**个人对JS中的函数与闭包理解的php版本**
可以调用的对象
	- 拥有operator()属性的对象,可以被调用
	- 理解以下php代码
```php
<?php
class Func
{
	public $property = "i'm property\n";

	public function method()
	{
		return "i'm method\n";
	}

	// 等同于重载 operator ()
	public function __invoke()
	{
		return "i'm invoked\n";
	}
}

$func = new Func();
echo $func->property; // i'm property
echo $func->method(); // i'm method
echo $func(); // i'm invoked

$freeVar = "i'm refed free var\n";
$anotherFunc = function() use($freeVar) {
	return $freeVar;
}; // 注意分号,,此处为表达式(lambda)

echo $anotherFunc(); // i'm refed free var

// 所以,php无闭包? 显式use
// php 匿名函数模块化 ?!

```

创建函数的两种方式:

example:
``` javascript
// 在浏览器全局环境中创建函数

// 常规方式
function square() {
	return x * x
}

// or 对匿名函数求值[,赋值到变量]
var square = function(x) {
	return x * x
}
```

区别:
1. 匿名函数方式可以明确表示square是一个包含函数值的变量
2. function表达式方式解析时会被提升,无论何处定义,都会被移动到函数作用于顶层
同时导致在if语句使用function表达式中创建函数不同浏览器存在歧义
~~~
if(false) { function func(){} }
console.inf(func) // ?
~~~


## 匿名函数
lambda其实就是 x 到 y 的映射关系, 但在大部分支持函数式的编程语言中, 它等价于匿名函数. 被称为 lambda 表达式. 因为这些函数只需要用一次, 而且变换不复杂, 完全不需要命名
匿名函数在程序中的作用是可以作为参数传给高阶函数, 或者作为闭包被返回.

~~~
匿名函数并不是原本的 lambda 算子, 因为匿名函数也可以接受多个参数.
multiple(x, y) = x*y

写成简单映射的形式, 把名字去掉
(x,y) -> x*y
这就是 lambda 了吗, 不是, lambda的用意是简化这个映射关系以至不需要名字, 更重要的是只映射一个 x.

什么意思呢? 让我们来分解一下上面的这个映射的过程.

lambda 接受第一个参数 5, 返回另一个 lambda
(5) -> (y -> 5*y)

该返回的 lambda y -> 5*y 接收 y 并且返回 5*y, 若在用 4 调用该 lambda
es6 -> 5*4

因此这里的匿名函数 (x,y)->x*y 看似一个 lambda, 其实是两个 lambda 的结合.

而这种接受一个参数返回另一个接收第二个参数的函数叫柯里化3.

~~~

JavaScript 中的 lambda 表达式


高阶函数
~~~
匿名的一等函数到底有什么用呢? 来看看高阶函数的应用.

高阶函数意思是它接收另一个函数作为参数. 为什么叫 高阶: 来看看这个函数 f(x, y) = x(y) 按照 lambda 的简化过程则是

f(x) => (y -> x(y))
(y) => x(y)
可以看出来调用 f 时却又返回了一个函数x.

还记得高等数学里面的导数吗, 两阶以上的导数叫高阶导数. 因为求导一次以后返回的可以求导.

概念是一样的, 如同俄罗斯套娃 当函数执行以后还需执行或者要对参数执行, 因此叫高阶函数.


高阶函数最常见的应用如 map, reduce. 他们都是以传入不同的函数来以不同的方式操作数组元素.

另外 柯里化, 则是每次消费一个参数并返回一个逐步被配置好的函数.

高阶函数的这些应用都是为函数的组合提供灵活性. 在本章结束相信你会很好的体会到函数组合的强大之处.
~~~