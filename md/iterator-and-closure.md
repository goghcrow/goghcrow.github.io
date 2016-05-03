# 迭代器与闭包

发现迭代器可以理解为: **闭包对象**

~~~ javascript
"use strict"

const iterMin = min => () => min++
const iterMinMax = (min, max) => () => min <= max ? min++ : void 0
const iterArray = array => {
	let i = 0
	return () => array[i++]
}

!(() => {
	let ret = []
	for (let iter = iterMin(0), i = iter(); i < 10; i = iter()) {
		ret.push(i)
	}
	console.dir(ret)
	// [ 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 ]
}())

!(() => {
	let ret = []
	for (let iter = iterMinMax(0, 9), i = iter(); i < 10; i = iter()) {
		ret.push(i)
	}
	console.dir(ret)
	// [ 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 ]
}())

!(() => {
	let array = [0,1,2,3,4,5,6,7,8,9]

	let ret = []
	for (let iter = iterArray(array), i = iter(); i < 10; i = iter()) {
		ret.push(i)
	}
	console.dir(ret)
	// [ 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 ]
}())

~~~


~~~ php
<?php
function iter($min) {
	return function() use(&$min) {
		return $min++;
	};
}

function generator($min, $max) {
	while($min <= $max) {
		yield $min++;
	}
}

$ret = [];
for ($iter = iter(0), $i=$iter(); $i < 10; $i = $iter()) {
	$ret[] = $i;
}
$gen = generator(0, 9);
assert(iterator_to_array($gen) === $ret);
~~~