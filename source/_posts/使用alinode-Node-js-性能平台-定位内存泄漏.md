---
layout: title
title: 使用alinode(Node.js 性能平台)定位内存泄漏
date: 2019-04-22 12:15:33
tags: [运维,DEBUG]
categories: 技术
---
## 背景
few项目占用内存居高不下，而且随着时间推移越来越高。刚启动的时候few所占内存仅100MB左右(如图一)，随着时间推移，内存一直上升，甚至到达1G(如图二)。基本确定是内存泄漏

![B33A9687-FE24-426F-8D0C-BCDDF1BF7A](http://cdn.jsblog.site/B33A9687-FE24-426F-8D0C-BCDDF1BF7AC8.png)

![9E4C9CBB-0281-4CB2-AA27-0A70A02AF132](http://cdn.jsblog.site/9E4C9CBB-0281-4CB2-AA27-0A70A02AF132.png)


## 接入alinode
Egg项目接入alinode极其简单，egg官方文档有对应的介绍[应用部署 - 为企业级框架和应用而生](https://eggjs.org/zh-cn/core/deployment.html#nodejs-%E6%80%A7%E8%83%BD%E5%B9%B3%E5%8F%B0alinode)。

应用到我们自己的项目仅需三步：
1. 在阿里云开通`Node.js 性能平台`(免费)
2. 项目内安装`egg-alinode`依赖，开启插件并且进行配置
3. 在项目内安装`AliNode Runtime`替换`Node.js Runtime`，这个直接在`Dockerfile`里把基础镜像改成对应的`alinode`镜像即可

运行你的项目，就可以在alinode控制台看到你的应用上线了。

## 开始诊断
![62159F8F-EC0A-4B52-AB07-5CA22B95E3B6](http://cdn.jsblog.site/62159F8F-EC0A-4B52-AB07-5CA22B95E3B6.png)


第一步先生成一份初始堆快照，然后等待程序运行一段时间（时间越长越容易看出问题），再次生成第二份快照。

![7F2FF0B0-0F1D-4E41-A88E-5AFB11997345](http://cdn.jsblog.site/7F2FF0B0-0F1D-4E41-A88E-5AFB11997345.png)


将两份文件均进行转储，并将其下载到本地。

![F0D42F21-7FFC-4677-9802-1849D71DF71B](http://cdn.jsblog.site/F0D42F21-7FFC-4677-9802-1849D71DF71B.png)


进入devtools，将两份文件上传进行比较。这个时候就可以看到在这两个时间点之间，到底堆内存发生了哪些变化。

经过简单的排查，发现有大量重复的对象，里面含有`debug``mongo`等内容，所以将目标锁定在`co-common`内引用的`mongo`模块。因为不想在上面花费过多时间，而且`few`项目本身并不需要`mongo`，所以直接对`co-common`做了精简，只保留了需要的两个方法。（如果谁感兴趣的话，可以看下mongo模块和debug 模块的源码，现在了解到的信息是debug模块存在闭包，而mongo模块引用了debug模块且很可能没有正常销毁）

改完后重新上线观察，效果喜人，不仅内存没有持续增加，而且由于去掉了不必要的依赖，初始状态占用内存低了不少。

## 最后
1. 还有其他项目依赖了`co-common`，但依赖程度比`few`高，需要逐步移除
2. 后续的开发过程中，依赖从简，不需要的坚决不要引入
3. alinode性能平台上，除了可以定位内存泄漏问题，还有其他一些运维功能，比如`qps`、`接口慢查询`等。


