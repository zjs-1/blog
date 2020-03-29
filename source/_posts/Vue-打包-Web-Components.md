---
title: Vue 打包 Web Components
date: 2020-03-29 21:52:26
tags: [技术,前端]
categories: 技术
---

# Vue 打包 Web Components

## 背景介绍

公司有一个旧项目，技术栈为PHP + bootstrap + jquery + seajs，该项目承载了很多业务逻辑。由于技术栈比较旧，在开发上遇到一些问题，比如模块化、可交互性上都不是很好。
针对管理后台，又新开了一个项目，技术栈为node + vue，解决了以前许多开发上的问题，开发效率也高很多。但是由于旧项目内容较多，并没有一次性把所有内容进行迁移，而是业务驱动，逐渐迁移。

当前有个需求迭代，在进行综合评估后(功能全部迁移耗时较长，而且该业务后续可能会有较大调整)，决定在旧项目上进行开发。
该迭代内，需要新增一个编辑器，该编辑器需要一定的数据交互性，而且后续功能迁移至新项目后该编辑器还是需要使用的。经过简单评估之后，决定使用vue来编写该编辑器，并且编译成Web Components在旧项目内使用。

> 该项目是管理后台项目，为内部人员使用，对浏览器兼容性要求较低，用户端不建议这样使用

## 编写组件
https://cli.vuejs.org/zh/guide/prototyping.html ，使用vue直接开发组件。因为是第一次使用，所以过程中先写了一些demo，验证可行性，确定没问题之后才放开手进行开发。

## 打包

https://cli.vuejs.org/zh/guide/build-targets.html#web-components-%E7%BB%84%E4%BB%B6

使用vue-cli将编写好的组件打包成Web Component

## 使用

上一步打包完之后，会得到一个组件的JS文件，在项目内引入该文件和`vue`文件，然后就可以在`HTML`中直接使用了，如下

```
<script src="https://cdn.bootcss.com/vue/2.6.11/vue.js"></script>
<script src="/bundles/topxiaweb/js/controller/task-type-manage/richtext-editor.min.js"></script>
<body>
    <richtext-editor></richtext-editor>
</body>
```

## 遇到的问题

上面过程看起来很简单，但是实操过程还是遇到了一些问题，以下作为记录:

### 引入后，提示找不到Vue

原因是vue.js引入顺序有问题，没有在组件文件之前引入，但是script书写顺序是正确的。主要是旧项目框架的问题，没什么代表性，不细讲，总之遇到了就先检查文件加载顺序。
    
### 第三方组件CSS加载异常

例如`vue-quill-editor`组件，其引入css的方式如下:

```javascript
import 'quill/dist/quill.core.css'
import 'quill/dist/quill.snow.css'
import 'quill/dist/quill.bubble.css'
import { quillEditor } from 'vue-quill-editor'
export default {
  components: {
    quillEditor
  }
}
```

在网上查询之后，改变了引入的方式，用到了相对路径，感觉不是特别优雅，后续可以找优化方式，主要就是改成css的@import，如下:

```
<style scoped>
  @import '../node_modules/quill/dist/quill.core.css';
  @import '../node_modules/quill/dist/quill.snow.css';
  @import '../node_modules/quill/dist/quill.bubble.css';
</style>
```

### 字体文件没有加载

在使用`element-ui`的图标时，发现字体文件没有正常加载，这个目前没有找到原因，解决方式就是把字体文件的引入放在外部去做。在这个场景里，就是在旧项目里直接引入字体文件，如下:

```
<style type="text/css">
    @font-face {
      font-family: element-icons;
      src: url("https://unpkg.com/element-ui@2.13.0/lib/theme-chalk/fonts/element-icons.woff") format("woff"),url("https://unpkg.com/element-ui@2.13.0/lib/theme-chalk/fonts/element-icons.tff") format("truetype")
    }
</style>
```

### 组件数据传递

- 父级向组件传递数据，可直接通过attr进行传递，如下两种方式都可以，可根据具体情况选用对应的方式，需要注意的是，如果直接在`HTML`里进行传递`&quot;`会被识别成`"`，可能会出现出乎意料的bug，尤其在传递富文本的时候。传递复杂对象时，目前看到的比较好的方式就是先做JSON字符串化进行传递，组件内再做解析。

```
<richtext-editor class="test" params='{"a": 1}'></richtext-editor>

$('.test').attr('params', '{"a": 1}');
```

- 组件接收数据，在`vue`里面就是直接用props接收就可以了，这一点还是很方便的。
- 组件向父级传递数据，就是使用vue的`this.$emit()`，父级监听对应的事件，就可以从事件内取到对应的数据，如下:

```javascript
$(".question-editor").on("change", function(e) {
    let data = JSON.parse(e.originalEvent.detail);
});
```

## 看下成果

红框部分就是这次的主要内容，看看这`bootstrap`和`element-ui`的混搭风格

![](http://cdn.jsblog.site/15854896795167.jpg)



