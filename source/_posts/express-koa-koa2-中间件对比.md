---
title: express & koa & koa2 中间件对比
date: 2018-07-21 14:10:23
tags:
---

## 背景

最近在做标签系统，使用`koa2`框架，期间尝试使用`swagger`来定义接口，在寻找`swagger validate`中间件的时候遇到了问题。

首先找到了[swagger-tools](https://github.com/apigee-127/swagger-tools)，项目拥有500+个star，很满意，但它是一个express风格的中间件，并不直接兼容`koa2`。一顿操作后，虽然能在`koa2`里使用了，但是却检验不了出参。整个过程后文再做描述吧。

接着又是一顿找，找到使用import风格的，目前线上的node版本是v7.9.0，明显也是不能直接用了，放弃。

然后又选择了[koa-swagger](https://github.com/rricard/koa-swagger)，看起来也很满足需求，虽然使用的是koa1的generator风格，但是只要稍作处理就可以在koa2中使用了，但比较介意的是star只有17，感觉可能会有些坑。果不其然，在项目进行到一半的时候，发现它里面有个方法在解析路径地址的时候有问题，地址里只允许携带一个参数，ORZ。

最后还是用回了swagger-tools，舍弃掉了出参判断。

在整个过程中，由于对三个框架的中间件风格并没有仔细研究过，导致走了好多弯路，所以决定以这个契机，好好研究下这三个框架中间件有什么区别。

## express 中间件

