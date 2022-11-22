---
layout: title
title: JS异步编程
date: 2020-03-29 22:32:07
tags: [Javascript, 技术分享]
categories: 技术
---

## 同步和异步
计算机领域里的`同步`和生活中的`同步`是不同的概念：
生活中的`同步`，比如后端和前端同步开发，这个在同步应该是同时的意思，在计算机领域里，应该和`并行`比较相似。

而计算机领域里的`同步`和`异步`又是什么意思呢？
同步就是所有过程顺序执行，A执行完毕才执行B，B执行完才执行C。
异步则不同，所有过程之间是可能并行执行的，比如执行A过程，但是A还没执行结束并返回结果，就已经开始执行B和C了。

举个例子
现在在开发某个需求，需要黑白提供一个接口，我搬了个小板凳坐在黑白旁边看着他，等他写完接口之后，我再开始拿接口来写页面，这就是`同步`；
如果在黑白写接口的同时，我就开始写页面样式，接口出完我直接对接，这就是异步；

## 进程和线程
为了进入下一部分，还是先简单讲下`进程和线程`，这部分不细讲，因为不是今天的重点，重要的是大学的时候没好好听，所以想讲也讲不了。

`进程和线程`又是计算机领域的概念了。
简单粗暴地理解，运行一个软件就是一个进程，比如我开了一个微信两个记事本，那就有一个微信进程、两个笔记本进程（当然了，一个软件不一定是单进程的，也可能是多进程的）。
那线程又是什么？因为有些进程同一时间不止做一件事情，比如微信，我在打字的同时还要能接收到别人的信息，那这就需要两个线程。

举个例子
一家快餐店可以看作一个进程，如果只有老板一个人(即为单线程)，他要一个人负责点单、炒菜、打包等任务，那最高效的情况下，他也只能一次服务一个客人，客人多了就得排队。
老板招了几个伙计，可以看作这个进程多开了几个线程，各做各的事，有人负责点单，有人负责炒菜，有人负责打包，那就可以同时服务很多个客人了。
当然了，如果老板有钱，那也可以在隔壁再开个分店，也就是弄一个新的进程，当然成本就稍高一些。

在计算机里也是一样的，实现多任务有三种方式：
1. 多进程
2. 多线程
3. 多进程 + 多线程

当然了，上面的例子不是很恰当，计算机执行的很多任务，只有CPU这个员工在做，八核的CPU那就是八个员工。
才八个员工，怎么同时做那么多事情呢，这就涉及到CPU的时间片和调度了，简单说就是把CPU资源按时间切分成片，比如10毫秒一片，那一秒就有100片了，把这些时间片以一定规则对线程进行轮转调度，就可以让使用者感觉多个线程之间在并行计算了。

## JS单线程、异步？
之前一直默认`JavaScript `(后简称JS)就是单线程、异步的，虽然模模糊糊地也知道这是什么个东西，但也说不明白。仔细想了想还有点奇怪，我们回想下前面`异步`和`线程`的概念，异步和单线程应该是矛盾的呀。

比如现在公司只有一台电脑可以用，可以看成是单线程的，那我和黑白怎么异步办公呢，我就必须得等黑白在电脑上把接口写好，我才能在电脑上接着写页面，这就是同步了。

我们先来看看JS的单线程指的是什么？JS是一门编程语言，需要在特定的虚拟机里面运行，这个虚拟机就叫`JavaScript引擎`，大家接触最多、最广的应该就是附带在网页浏览器里的JS引擎了，这个时候，我们称网页浏览器是JS的`宿主环境`。
现代浏览器内核有两个主要组成部分，一个是渲染引擎（负责将网页内容渲染展示），另一个就是JS引擎（负责解析、执行JS）。JS的单线程，主要是因为对应的JS引擎没有给它提供创建线程的能力，至于为什么不提供，主要是为了避免`线程竞争`等繁琐的问题。

既然JS是单线程的，为什么还能异步呢，因为宿主环境是多线程的。比如用JS发起一个网络请求，因为请求返回是要耗时的，不能因此就阻塞住后面的操作，那负责处理网络请求这个工作，就扔给浏览器对应的线程去处理就好了，代码可以继续往下执行，这就是异步。

## JS的异步编程
讲到这里，算是把背景知识介绍完了，开始进入正题，感觉正题可能比背景还短。

### 回调函数

回调函数是异步编程里最基本的方式，比如下面的代码:

```javascript
function division(a, b, sec, cb) {
	setTimeout(function() {
		if (Number(b) === 0) {
			cb(new Error('除数不能为0'));
		} else {
			cb(null, a/b);
		}
	  return 
	}, sec * 1000);
}

function main() {
	division(100, 10, 1, function(err, result) {
		if (err) {
			return console.error(err);
		}
		console.log(`100/10=`, result);
	});
	division(100, 0, 2, function(err, result) {
		if (err) {
			return console.error(err);
		}
		console.log(`100/0=`, result);
	});
}

main();
```

这是经典的回调函数的写法，`error-first`算是一种规范了，就是在回调函数中，错误放在第一个参数。

#### 回调地狱

```javascript
listen("click", function handler(e) {
	// 点击后等待一秒
	setTimeout(function request() {
		// 发起请求1
		ajax("url1", function(err, res1) {
			if (!err) {
				return console.error(err);
			}
			// 发起请求2
			ajax("url2", data, function(err, res2) {
				if (!err) {
					return console.error(err);
				}
				console.log(res2);
			});
		});
	}, 1000);
});
```

```javascript
doA(function() {
	doB();
	doC(function() {
		doD();
	});
	doE();
});
doF();
```

当大量的回调函数组合在一起的时候，很难直观地找到正确的执行顺序，我们可能更加习惯A=>B=>C这样的有序执行，所以需要刻意训练才有办法习惯这种异步回调的写法。

上面的例子还算是比较简单的，一些复杂的场景实现起来会更加难看，比如同时发起三个请求，都完成之后，再进行操作之类的。

#### 信任问题

之前一直以为`回调地狱`是回调函数的唯一问题，后来在《你不知道的JavaScript》中才了解到，还有一个信任问题。

```javascript
function getMessage(cb) {
	// setTimeout(function() {
	setInterval(function() {
		let message = {
			id: 1,
			data: 'Hello world',
		}
		cb(null, message);
	}, 1000);
}

function main() {
	let messages = [];
	getMessage(function(err, message) {
		if (err) return console.error(err);
		messages.push(message);
	});
}
```

上面的代码中，getMessage方法错误地多次调用回调函数，导致程序可能出现无法预料到的问题。虽然例子里的问题很容易发现，也很容易修复，但是实际的业务代码中问题可能藏得很深，而且有些函数是第三方提供的，并不是我们所能控制的。

信任问题即，回调函数可能被多次调用，或者过晚、过早调用。
问题的本质在于，回调函数这种方式，会把代码的执行控制权交给其他其他第三方，而这种控制反转是没有一份契约来明确表达行为的，比如什么时候执行回调，执行次数限制等。

### Promise

Promise是ES6新增的API，这里先不介绍API，先讲一个场景：

当我去喜茶买了杯喝的，因为饮料不是现成的，所以收银员给了我一个号码牌，告诉我叫号了再来拿。拿到这个号码牌，虽然我还没办法喝上这杯饮料，但对我来说一定程度上我已经拥有了这杯饮料，我已经可以用这个号码牌在朋友圈打卡了。而等到饮料做好之后，我就可以拿这个号码牌换到我的那杯饮料，当然也可能被告知“材料不够，饮料做不出来了”。

Promise就类似这个号码牌，针对异步操作，它可以给你提供一个“未来值”，你可以在一定程度上去使用这个“未来值”。

```javascript
function payForTea() {
	// 返回号码牌
	return new Promise(resolve => {
		setTimeout(() => {
			// 3s 后给你一杯绿茶
			resolve('A cup of green tea');
		}, 3000);
	});
}

// 分享给好友
function shareToFriend(myPromise) {
	myPromise.then(data => {
		console.log('Everybody look here, this is my:', data);
	});
}

// 买茶
let myTea = payForTea();

// 发条朋友圈
shareToFriend(myTea);
myTea.then(tea => {
	// 拿到茶了
	console.log('now i get:', tea);
});
```

> 简单介绍下API，不过多介绍：
 
![6E59A960-F16E-4A57-A00C-B1B6D780D1F5](media/15845496559554/6E59A960-F16E-4A57-A00C-B1B6D780D1F5.png)

#### 怎么解决信任问题

回调函数的信任问题主要是因为“控制反转”，在回调函数里写好操作之后，这些操作的调用就交由第三方控制了。
而在Promise里，第三方只是做结果通知，对通知结果的操作，还是由我们自己的代码控制，即把反转了控制反转。

举个例子，我现在要浇花，但是没有水，需要等黑白去帮忙打水。
如果是以前面的回调操作，就是我告诉黑白，“打完水之后，帮我浇下花，大概200毫升水”。这个时候这盆花的生死就完全交给了黑白了，他可能打完水之后，一下子给我的花浇了十次水，直接给淹死了。

而promise呢，就是我告诉黑白，“打完水，告诉我一声”。
那就只有三种可能，黑白告诉我打到水了，我自己去给花浇水，对应了promise里的`fulfilled`状态
又或者黑白告诉我，厕所没水了，那我就只能自己想办法了，对应了promise里的`rejected`状态
也可能黑白掉厕所里，再也没回来，程序就没有人那么聪明了，就会一直在那等着，除非设置了超时判断。这个时候promise一直处于`pending`状态

两种方式有什么不同呢，核心就在于，给花浇水这件事是我自己来执行，可以避免出错。而且promise限制了，状态改变之后就会凝固，也就是即使黑白告诉我十次他打完水了，我也不会傻傻地给花浇十次水，因为只有第一次通知是有效的。

#### 解决回调地狱

```javascript
function main() {
	requestA()
		.then(function() {
			requestB()
				.then(function() {
					requestC();
				});
		});
}

function main() {
	requestA()
		.then(function() {
			return requestB();
		})
		.then(function() {
			return requestC();
		}
		.then(function() {
			return requestD();
		};
}
```

如上图，promise的链式写法的确一定程度上可以缓解回调地狱，但是还是不够美观，看起来大量的`then`和`function`，配合上`箭头函数`可以进一步优化。
但还不称不上是最终解决方案，因为我们还是没办法直观从各种`then`之间找到代码的正确执行顺序。

```javascript
doA()
	.then(function() {
		doB();
		doC()
			.then(function() {
				doD();
			});
		doE();
	});
doF();
```

### generator函数

generator函数是ES6新增的语法，这里也不过多地介绍其语法的内容了，直接看例子。

```javascript
function* gen() {
	console.log('start Q&A');
	let a = yield 'question one:';
	console.log(a);
	let b = yield 'question two:';
	console.log(b);
	let c = yield 'question three:';
	console.log(c);
}

let i = gen();
console.log(i.next().value);
console.log(i.next('answer one').value);
console.log(i.next('answer two').value);
console.log(i.next('answer three').value);
```

Generator函数是一个`Iterable`对象，执行后返回一个迭代器，基于这个可以有不少玩法，但是今天不讲这个，我们主要看下，Generator是怎么处理异步的。

可以把generator简单地看成一个Q&A，内部提出一个问题后暂停，等到迭代器回答之后，才接着往下执行，直到提出下一个问题，等待下一个答案。

```javascript
let it = main();

function* main() {
	let res1 = yield ajax(url, null, function(res1) {
		it.next(res1);
	});
	let res2 = yield ajax(url2, res1, function(res2) {
		it.next(res2);
	});
}
```
```javascript
let it = main();
it.next();

function fetchNum(a, cb) {
	setTimeout(function() {
		cb(a*10);
	}, 1000);
}

function* main() {
	let res1 = yield fetchNum(1, function(res1) {
		it.next(res1);
	});
	console.log(res1);
	let res2 = yield fetchNum(res1, function(res2) {
		it.next(res2);
	});
	console.log(res2);
}
```

上面这段代码，使用回调+generator，虽然初看起来还是有点懵，但是细看之后，会发现这种写法有一个巨大的进步，我们可以用同步的写法进行异步编程了。但是把`迭代器it`写到回调函数里是有入侵性的，而且前面说的回调函数存在的问题还是存在。

下面我们试着把generator和promise做个结合。
```javascript
let it = main();
it.next();

function fetchNum(a) {
	return new Promise(resolve => {
		setTimeout(() => {
			resolve(a*10);
		});
	});
}

function* main() {
	let res1 = yield fetchNum(1);
	console.log(res1);
	let res2 = yield fetchNum(res1);
	console.log(res2);
}
```

很遗憾，跑不起来，差了点什么，仔细看，`it.next()`拿到一个promise，但是并没有对这个promise的未来值做任何操作。
这个时候感觉就像是自行车已经装好了，但是没有动力去推动它前进，你可以站上去一脚一脚地把它踩起来，也可以选择给它装一个自动引擎，让它变成无人驾驶电动车。

```javascript
co(main());

function co(it) {
	run(it.next());
	
	function run(step) {
		if (!step.done) {
			let p = step.value; // 这里假设所有yield后面跟的都是promise
			p.then(data => {
				run(it.next(data));
			});
		}
	};
}

function fetchNum(a) {
	return new Promise(resolve => {
		setTimeout(() => {
			resolve(a*10);
		});
	});
}

function* main() {
	let res1 = yield fetchNum(1);
	console.log(res1);
	let res2 = yield fetchNum(res1);
	console.log(res2);
}
```


