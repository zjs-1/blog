---
layout: title
title: 三十分钟学不会ES6
date: 2018-1-3 11:27:33
tags: [Javascript,技术分享]
categories: 技术
toc: true
---

> 本文仅仅是我在看`《ECMAScript6入门》阮一峰`这本书过程中的一个小结，不包含所有ES6的内容。如果对ES6真正感兴趣，推荐直接阅读[ECMAScript6入门](http://es6.ruanyifeng.com/#README)。
 
## 前言
ES6是ECMAScript 6.0的简称，是JavaScript这门语言的一个标准。
### ECMAScript与JavaScript
JavaScript诞生于1995年（我也是），由Netscape公司创造，作为浏览器的脚本语言，处理一些表单验证操作。随后微软在IE浏览器里也加入名为JScript的JavaScript实现。导致当时存在3个不同的JavaScript版本，3个不同版本并存会带来很多问题，因此Netscape决定将JavaScript提交给国际标准化组织ECMA，希望这门语言能成为国际标准。
1997年ECMA发布标准262号标准文件(ECMA-262)的第一版，规定了浏览器脚本语言的标准，并将这种语言称为ECMAScript。
用人话说就是，ECMAScript是JavaScript的规范，JavaScript是ECMAScript的一种实现
### ES6与ES2015
ES6是一个历史名词，是一个泛指，指的是5.1版本后的JavaScript的下一代标准，涵盖了ES2015、ES2016、ES2017等。本文提到的ES6部分，一般指ES2015标准。

## 变量声明命令
### let 命令
- 块级作用域
- 不存在变量提升
- 暂时性死区
- 不允许重复声明

### const 命令
- 与let上面四条特性
- 变量指向的内存地址不得改动

### 所有声明变量的方式
- var
- function
- let
- const
- import
- class


``` javascript
//块级作用域
{
	var a = 2;
	let b = 3;
	const c = 4;
}

console.log(a); // 2
console.log(b); // error
console.log(c); // error

//变量提升
console.log(a); //undefined
var a = 1;

console.log(b); //error
let b = 2;

console.log(c); //error
const c = 3;

//暂时性死区
a = 2;
console.log(a); //2
{
	a = 3; //error
	let a; 
	console.log(a); //undefined
}

//不允许重复声明
{
	var b = 10; //10
	var b = 20; //20

	let a = 10; //10
	var a = 20; //error
	let a = 20; //error
}

//const变量指向的地址不得改变
{
	const foo = {};
	foo.prop = 123;
	console.log(foo.prop) //123
	foo = {}; //error
}
```

## 变量的解构赋值
### 数组解构赋值

``` javascript
//以前
var arr = [1, 2, 3];
var a = arr[0];
var b = arr[1];
var c = arr[2];
//现在
let [d, e, f] = [1, 2, 3];
//甚至
let [g, [h, i], j] = [1, [2, 3], 4];
```

### 对象解构赋值
```javascript
let {a, b} = {a: 'aaa', b: 'bbb'};
console.log(a); //aaa
console.log(b); //bbb
let {a: c} = {a: 'aaa'}
console.log(c); //aaa
```

### 字符串解构赋值
```javascript
let [a, b, c] = 'zjs';
console.log(a); //z
console.log(b); //j
console.log(c); //s
```

### 数值和布尔值的解构赋值
```javascript
let {toString: s} = 123;
s === Number.prototype.toString // true

let {toString: s} = true;
s === Boolean.prototype.toString // true
```

### 函数参数解构赋值
```javascript
function updateUser({id, name, height}) {
	//blablabla
}
let user = {
	id: 1,
	name: 'zjs',
	height: 180
};
updateUser(user);
```

## 字符串的扩展
> 针对双字节Unicode字符的改进此文跳过

### indexOf方法的扩展
```javascript
var s = 'Hello world!';

s.startsWith('Hello') // true
s.endsWith('!') // true
s.includes('o') // true
```

### repeat方法
```javascript
'我爱你\n'.repeat(100); //快速输出100行我爱你
```

### padStart和padEnd
```javascript
function rate(num, maxNum) {
	return '★'.repeat(num > maxNum ? maxNum : num).padStart(maxNum, '☆');
}
//快速实现一个星级评分
```

### 模版字符串
```javascript
//以前
var content = 'hello';
var a = '<p>' +
	'<span>' + content + '</span>' +
	'</p>';
	
//现在
let a = `<p>
	<span>${content}</span>
	</p>`;
```
> 关于模版字符串还有不少内容，最好直接看书了解

## 正则的扩展
> 对于正则不够熟悉，觉得正则的内容应该再重新过一遍，简单的说：ES6增加了u修饰符以适应双字节的Unicode字符，y修饰符作为粘连匹配，s修饰符使得小数点可以匹配任意单个字符甚至行终止符（提案），支持后行断言（提案）。

## 数值的扩展
> 将全局方法如parseInt()、parseFloat()等移至Number对象下，目的是减少全局性方法，使语言模块化。
> 
> 增加了许多数学方法，可以自行了解。

## 函数的扩展
### 参数默认值
```javascript
//以前
function test(a, b) {
	a = a || 'hello';
	b = b || 'world';
	return a + ' ' + b;
}
//	现在
function test(a = 'hello', b = 'world') {
	return a + ' ' + b;
}
//实用用法
function throwMissingError() {
	throw new Error('Missing parameter');
}
function test(a = throwMissingError()) {
	return a;
}
```

### reset参数
```javascript
function add(...values) {
  let sum = 0;

  for (var val of values) {
    sum += val;
  }

  return sum;
}

add(2, 5, 3) // 10
```

### name属性
```javascript
function a() {};
a.name; //'a'
```

### 箭头函数
```javascript
//以前
function sum(a, b) {
	return a+b;
}
//现在
sum = (a, b) => a + b;
square = n => n * n;
isEven = n => n % 2 == 0;
```
> 箭头函数还有许多要注意的地方，详细看书

### 尾调用优化和函数式编程
> 比较晦涩但又觉得有趣、有用的一块内容，此处不讲，也讲不了。

## 数组的扩展

### 扩展运算符
扩展运算符（spread）是三个点（...）。它好比 rest 参数的逆运算，将一个数组转为用逗号分隔的参数序列。

```javascript
console.log(1, ...[2, 3, 4], 5)
// 1 2 3 4 5
```

### fill()
```javascript
//数组初始化
let array = new Array(100);
array.fill('hello');
```

### 数组实例的 entries()，keys() 和 values()
```javascript
for (let index of ['a', 'b'].keys()) {
  console.log(index);
}
// 0
// 1

for (let elem of ['a', 'b'].values()) {
  console.log(elem);
}
// 'a'
// 'b'

for (let [index, elem] of ['a', 'b'].entries()) {
  console.log(index, elem);
}
// 0 "a"
// 1 "b"
```

## 对象的扩展

## 新的原始数据类型Symbol
> JavaScript之前有六种数据类型: undefined、null、布尔值、字符串、数值、对象
> 新增了Symbol类型

### 声明
```javascript
let s = Symbol('a');
typeof s; //'symbol'
s.toString(); // 'Symbol(a)'
```

### 独一无二
```javascript
let s1 = Symbol('a');
let s2 = Symbol('a');
s1 === s2 //false
```

### 用途
有些时候我们可能会定义一些常量，但是内容我们并不关心，比如:

```javascript
const shapeType = {
	circle: 'circle',
	triangle: 'triangle',
	square: 'square'
}
switch (type) {
	case shapeType.circle:
		//blablabla;
		break;
	case shapeType.triangle:
		//blablabla;
		break;
	case shapeType.square:
		//blablabla;
		break;
}
//可以改成
const shapeType = {
	circle: Symbol(),
	triangle: Symbol(),
	square: Symbol()
}
switch (type) {
	case shapeType.circle:
		//blablabla;
		break;
	case shapeType.triangle:
		//blablabla;
		break;
	case shapeType.square:
		//blablabla;
		break;
}
```

### Symbol.for(), Symbol.keyFor()
```javascript
var s1 = Symbol.for('foo');
var s2 = Symbol.for('foo');

Symbol.keyFor(s1) // "foo"
s1 === s2 // true
```

## Set和Map数据结构
### Set
Set类似于数组，但是成员值唯一
```javascript
// 去除数组的重复成员
[...new Set(array)]
```

```javascript
//Set的操作方法

var s = new Set();
s.add(1).add(2).add(2);
// 注意2被加入了两次

s.size // 2

s.has(1) // true
s.has(2) // true
s.has(3) // false

s.delete(2);
s.has(2) // false

s.clear();
s.size //0

```

```javascript
//Set的遍历方法

let set = new Set(['red', 'green', 'blue']);

for (let item of set.keys()) {
  console.log(item);
}
// red
// green
// blue

for (let item of set.values()) {
  console.log(item);
}
// red
// green
// blue

for (let item of set.entries()) {
  console.log(item);
}
// ["red", "red"]
// ["green", "green"]
// ["blue", "blue"]

set.forEach((value, key) => console.log(value) )
// red
// green
// blue

```

### Map
> Map与Object一样都是键值对的集合(Hash结构)，但是Object的键只能是字符串，Map的键可以是各种类型的值，实际上是一种“值-值”对应。

```javascript
//用法
const map = new Map([
  ['name', '张三'],
  ['title', 'Author']
]);

map.set('height', 180);

map.size // 3
map.has('name') // true
map.get('name') // "张三"
map.has('title') // true
map.get('title') // "Author"

map.delete('name');
map.has('name') //false

map.clear();
map.size; //0
```

```javascript
//遍历方法
const map = new Map([
  ['F', 'no'],
  ['T',  'yes'],
]);

for (let key of map.keys()) {
  console.log(key);
}
// "F"
// "T"

for (let value of map.values()) {
  console.log(value);
}
// "no"
// "yes"

for (let item of map.entries()) {
  console.log(item[0], item[1]);
}
// "F" "no"
// "T" "yes"

// 或者
for (let [key, value] of map.entries()) {
  console.log(key, value);
}
// "F" "no"
// "T" "yes"

// 等同于使用map.entries()
for (let [key, value] of map) {
  console.log(key, value);
}
// "F" "no"
// "T" "yes"
```

### WeakSet和WeakMap
> 与Set和Map一样，不同点在于WeakSet的成员以及WeakMap的键只能是对象，而且这两个对象都是弱引用，即不计入垃圾回收机制，无需手动清除。

## Promise对象
> Promise是异步编程的一种解决方案，比传统的解决方案(回调函数和事件)更合理和强大。

### Promise对象的特点
- 对象状态不受外界影响，有三种状态:pending、fulfilled、rejected，只有Promise里的操作才可以改变这个状态
- 状态一旦改变，就凝固了，不会再变。

### 基本用法
```javascript
var promise = new Promise(function(resolve, reject) {
	asynFunction(param, function(err, value) {
		if (err) {
			reject(err);
		};
		resolve(value);
	})
}

promise.then(value => {
	//blabla
}).catch(error => {
	//blabla
});
```

### 链式写法
Promise的then方法返回的是一个新的Promise实例，因此可以采用链式写法。

```javascript
var promise = new Promise(function(resolve, reject) {
        setTimeout(() => {
                resolve(1);
        }, 1000);
})

var promise1 = new Promise((resolve, reject) => {
        setTimeout(() => {
                resolve(2);
        }, 2000);
});

promise.then(value=>{
        console.log(value); //第一秒出现输出1
        return promise1;
}).then(value => {
        console.log(value); //第二秒输出2，注意是第二秒，promise在定义的时候就开始运行了
});
```

### 错误处理
catch方法可以捕获Promise对象抛出的错误以及前面then方法出现的错误。如果没有用catch方法捕获错误，外部将获取不到该错误。
catch方法返回的同样是一个Promise对象，支持链式写法。

```javascript
var promise = new Promise(function(resolve, reject) {
        throw new Error('错误1'); //后面的不会执行
        setTimeout(() => {
                resolve(1);
        }, 1000);
})

var promise1 = new Promise((resolve, reject) => {
        setTimeout(() => {
                resolve(2);
        }, 2000);
});

promise.then(value=>{
			//不会执行
        throw new Error('错误2'); 
        console.log(value);
        return promise1;
}).catch(error => {
        console.log(error); //输出错误
        return promise1;
}).then(value => {
        console.log(value); //2
});
```

### 其他方法
- Promise.all() 接收一个promise数组，全部resolve才resolve
- Promise.race() 接受一个promise数组，第一个resolve则resolve，第一个reject则reject
- Promise.resolve()与Promise.reject() 将现有对象转为Promise对象

---

#告一段落

> 以下是觉得比较重要但是没来得及准备的内容，如果已经觉得差不多了后面的就先不讲了，如果对下面哪部份感兴趣，可以现场讨论下。

## Iterator和 for...of循环

## Generator函数

## Class

## Decorator

## Module



