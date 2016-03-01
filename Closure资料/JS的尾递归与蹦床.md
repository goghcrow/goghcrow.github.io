JS 递归-尾递归优化TOC
1. 递归的问题
~~~
//使用递归将求和过程复杂化
var sum = function(x, y) {
    if (y > 0) {
      return sum(x + 1, y - 1)
    } else {
      return x
    }
}

sum(1, 10) // => 11

sum(1, 100000)
// Uncaught RangeError: Maximum call stack size exceeded
~~~

2. 尾调用优化 Tail Call Optimisation

尾调用概念就是在函数最后一步调用其他函数,尾递归即在最后一步调用自身

http://www.ruanyifeng.com/blog/2015/04/tail-call.html

执行尾递归时,程序无须储存之前调用栈的值,直接在最后一次递归中输出函数运算结果,这样就大大节省了内存，,这种优化逻辑就是在代码执行的时候将其转换为循环的形式

~~~
var sum = function(x, y) {
    var recur = function (a, b) {
        if (b > 0) {
            return recur(a + 1, b - 1)
        } else {
            return a
        }
    }

	//尾递归即在程序尾部调用自身，注意这里没有其他的运算
    return recur(x, y)
}

sum(1, 10); // => 11
~~~

3. Trampolining
~~~
var trampoline = function(f) {
    while (f && f instanceof Function) {
        f = f()
    }
    return f
}

var sum = function(x, y) {
    var recur = function(x, y) {
        if (y > 0) {
          return recur.bind(null, x + 1, y - 1)
        } else {
          return x
        }
    }
    return trampoline(recur.bind(null, x, y))
}

sum(1, 10) // => 11
sum(1, 100000)
~~~

trampoline函数接受一个函数作为参数,如果参数是函数就被执行后返回,如果参数不是函数将被直接返回，嵌套函数recur中，当y>0时返回一个参数更新了的函数，这个函数被转入trampoline中循环，直到recur返回x，x不是函数于是在trampoline中被直接返回。原文中作者对于每一步都有详尽的解释， 感兴趣的同学建议可以去看看原文。简单地说：以上逻辑就是将递归变成一个条件， 而外层trampoline函数执行这个条件判断并循环。

虽然解决了大参数递归的问题，但是却需要将代码转换成trampoline的模式，比较不灵活

不修改源码
~~~

var tco = function (f) {
    var value
    var active = false
    var accumulated = []

    return function accumulator() {
        accumulated.push(arguments)

        if (!active) {
            active = true

            while (accumulated.length) {
				// active = true , f.apply 返回 undefined
				// 且通过调用 f 往 accumulated中push了参数
                value = f.apply(this, accumulated.shift())
            }

            active = false

            return value
        }
    }
}

//这种方式确实有点奇怪,但的确没有改动很多源码,只是以直接量的形式使用tco函数包裹源码
var sum = tco(function(x, y) {
    if (y > 0) {
	  // 此处闭包, sum指向 tco返回的accumulator
      return sum(x + 1, y - 1)
    }
    else {
      return x
    }
})

sum(1, 10) // => 11
sum(1, 100000) // => 100001 没有造成栈溢出

~~~
