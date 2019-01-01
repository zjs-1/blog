---
title: 网页svg转png
date: 2018-07-06 16:39:29
tags: [前端]
categories: 技术
---

## 背景

在网页上看到一个logo想要保存下来，却发现右键不能下载，打开开发者工具发现这是一个svg标签，怎么样把它下载保存成png呢？

![源svg](http://cdn.jsblog.site/WX20180706-164613.png)

## 方法

```javascript
window.onload = function(){
	var svgString = new XMLSerializer().serializeToString(document.getElementById('svg'));

	var svg = new Blob([svgString], {type: "image/svg+xml;charset=utf-8"});
	var DOMURL = self.URL || self.webkitURL || self;
	var url = DOMURL.createObjectURL(svg);

    // 1. 将svg转成img标签(此时已经可以右键保存，使用在线工具转成png了)
	var img = new Image();
	img.src = url;
	document.getElementsByTagName('body')[0].appendChild(img);

    // 2. 使用img绘制canvas
	img.onload = function() {
		var canvas = document.getElementById('canvas');  //准备空画布
		canvas.width = img.width;
		canvas.height = img.height;
	 
		var context = canvas.getContext('2d');  //取得画布的2d绘图上下文
		context.drawImage(img, 0, 0);

		document.getElementsByTagName('body')[0].appendChild(convertCanvasToImage(canvas));
	}

	// 3. 将canvas转成图片
	function convertCanvasToImage(canvas) {
		var newImg = new Image();
		newImg.src = canvas.toDataURL("image/png");
		return newImg;
	}
}
```

![大功告成](http://cdn.jsblog.site/1530867968.jpg)


