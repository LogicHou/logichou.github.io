---
title: "HTML5 笔记"
date: 2022-09-22T20:28:17+08:00

categories:
 - 前端
tags:
 - html

draft: true
toc: true
---

# HTML5 的几个特性

## 空白折叠现象

文字和文字之间的多个空格、换行会被折叠成一个空格

标签“内壁”和文字之间的空格会被忽略

```html
<body>
  <p>文字1

               文字2</p> 文字1和文字2之间的空格和换行会被折叠成一个空格
  <p>   文字3   </p>     文字3前后的空格会被忽略
</body>
```

## 常见转义字符

```html
&lt;    小于号
&gt;    大于号
&nbsp;  空格（不会被折叠）
&copy;  版权符号©
``` 

## HTML 注释

```html
<!-- page header -->
<!-- page content -->
<!-- page footer -->
``` 

# 标签

HTMl 叫做“超文本标记语言”，超文本标记就是标签，不同的标签有不同的功能，一般都是成对出现的

## 原标签

原标签是单标签，用来做一些网页的基础配置

```<meta charset="UTF-8">``` 设置网页字符集

## title

```<title>我的网站</title>``` 设置网页标题

## 网页关键词和页面描述，方便 SEO

```html
<meta name="keywords" content="苹果官网, Apple官网, 苹果中国, Apple中国" />
<meta name="Description" content="探索 Apple 充满创新的世界，选购各式 iPhone、iPad、Apple Watch 和 Mac，浏览各种配件、娱乐产品，并获得相关产品的专家支持服务。" />
```

## div 标签

div 是英语 division “分割” 的缩写，**用来将相关内容组合到一起，以和其他内容分割**，使文档结构更清晰。比如网页头部、轮播图、文章内容要放入到一个 div 标签对中

div 像是一个容器，什么都可以容纳，也习惯将 div 称为“盒子”

## 标题标签

```html
h1  一级标题
h2  二级标题
h3  三级标题
h4  四级标题
h5  五级标题
h6  六级标题
```

# HTML5 骨架

在 vscode 中的空白 html 文件中输入一个 ! 号再按 tab 键盘，可以快速生成 HTML5 骨架代码

```html
<!DOCTYPE html>     <!-- 文档类型声明DTD -->
<html lang="en">    <!-- 表示网页的语言，zh表示中文，不改也行 -->
  <head>            <!-- head标签对用于网页的配置 -->
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
  </head>
  <body></body>     <!-- body标签对里面是网页真正主要的内容部分 -->
</html>
```

## 文档类型声明 DTD

* HTML 文件第一行必须是 DTD(Document Type Definition 文档类型声明)
* 不写DTD会引发浏览器的一些兼容问题
* 不同版本的 HTML 有不同的 DTD 写法

## div 常见的类名

div 标签可以添加 class 属性表示“类型”，类名服务于 CSS

```
区域          类名
------------------------
页头          header
logo         logo
导航条        nav
横幅          banner
内容区域       content 
页脚          footer
```