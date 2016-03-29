# php 数组内部指针

php5.x版本，之前对数组的理解是，hashtable + linkedtable的混合结构，linkedtable用于按序遍历；

[**current-doc**](http://php.net/manual/en/function.current.php)
> Every array has an internal pointer to its "current" element, which is initialized to the first element inserted > into the array.

同事写的代码昨天上线（线上环境php5.5），今天遇到一个bug，代码结构简化如下，

原始版本：
~~~ php
$arr = [1, 2, 3, 4, 5];
// 获取$arr的第一个元素
current($arr);
~~~

昨天上线版本：
~~~ php
$arr = [1, 2, 3, 4, 5];
// 业务变化，添加逻辑
foreach($arr as $val) {
	if($predicateExp) {
		break;
	}
}

// 获取$arr的第一个元素
current($arr);
~~~

bug描述：无法获取$arr的第一个元素;

同事喊我，瞄了一眼代码，然后跟她说不能用current获取数组第一个元素，因为当时不确定internal point在哪，

而且上方的foreach会改变internal point；

说完自己写了代码测试：
~~~php
$arr = [1, 2, 3, 4, 5];
foreach($arr as $v) {
}
var_dump(current($arr)); // out: int 1
~~~

晕~!，难道因为foreach是对数组copy一份来遍历的原因？！明明记得foreach会首先reset数组，重置interal point，之后进行遍历~
**wtf?**

于是又跟同事解释说，foreach不改变internal point；之后仍不安心，觉得自己没记错；看手册~

[**foreach-doc**](http://php.net/manual/en/control-structures.foreach.php)

> foreach (array_expression as $value)
>     statement
> foreach (array_expression as $key => $value)
>     statement
>     
> The first form loops over the array given by array_expression. 
> On each iteration, the value of the current element is assigned to $value 
> and the internal array pointer is advanced by one 
> (so on the next iteration, you'll be looking at the next element).
> 
> The second form will additionally assign the current element's key to the $key variable on each iteration.


** and the internal array pointer is advanced by one **

看来我原来的知识是正确的；

but:

> Note:
> In PHP 5, when foreach first starts executing, the internal array pointer is automatically reset to the first > > > element of the array. This means that you do not need to call reset() before a foreach loop.
> As foreach relies on the internal array pointer in PHP 5, changing it within the loop may lead to unexpected > > > behavior.
> In PHP 7, foreach does not use the internal array pointer.

打开[**3v4l**](https://3v4l.org/Yqcb7)，执行如下代码
~~~ php
$arr = [1, 2,3];
foreach ($arr as $key => $value) {
	echo "in foreach: ", current($arr) , PHP_EOL;
}
echo PHP_EOL;
var_dump("end foreach", current($arr));
~~~
<pre>
Output for 7.0.0 - 7.0.4, hhvm-3.8.1 - 3.12.0
in foreach: 1
in foreach: 1
in foreach: 1

string(11) "end foreach"
int(1)

Output for 5.5.0 - 5.6.19
in foreach: 2
in foreach: 2
in foreach: 2

string(11) "end foreach"
int(2)
</pre>

好吧，php7为了data-localized，修改了数组的实现，so，一并把数组的遍历实现也改了？
最近都在写java，php基本没关注~ 知识又落后了~

p.s.：
foreach的中文文档界面并木有提php7版本的问题~ 骚年，以后要看英文文档！！