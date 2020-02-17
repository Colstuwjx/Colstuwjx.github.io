---
title: "运维开发探索之< Gaea.03 前端 >"
date: 2015-04-19T20:00:00+08:00
tags: ["devops", "thinking"]
categories: ["devops"]
comments: false
showMeta: false
showAction: false
---

这篇文章是运维开发探索系列之Gaea篇的第三篇。

<!--more-->

#### 前置导读

如果你还未读过鄙人撰写的此系列文章的前面几篇的话，那么赶紧先去瞧一瞧吧：

传送门：[运维开发探索 开篇](http://devopstarter.info/devops-discovering-series_01)

传送门：[运维开发探索 Gaea.01 初探](http://devopstarter.info/devops-discovering-series_02)、[运维开发探索 Gaea.02 原型设计](http://devopstarter.info/devops-discovering-series_03)

这是运维开发探索系列之Gaea篇的第三篇，在这篇文章里，鄙人将会谈及关于运维平台工具开发的前端展示方面的一些个人见解。

#### MVVM? MVC?

前端在近几年可以说得到了爆发性的创新和迭代，出现了大量的创新理念以及技术框架，比如各种UI组件：[Bootstrap](http://getbootstrap.com/)、[Semantic-UI](http://semantic-ui.com/)，比如各种前端框架：[AngularJS](https://angularjs.org/)、[Meteor](https://www.meteor.com/)、[React.JS](https://facebook.github.io/react/)、[Polymer](https://www.polymer-project.org/) 等等。

总的来说，前端的变化在于从以前的前后端MVC（比如Django的Template理念便是前后端MVC的典型设计）转变到了前端MVC（或者有的人说是前端MVVM）。

关于前端MVC ( MVVM ) 的理念，鄙人觉得廖雪峰先生的博客讲解的十分到位，想详细了解的朋友可以转到：[廖雪峰的Python教程](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/0014025559313061080444477fc4b89a283944ecd922a74000)

前端MVC的理念主张的是“前后端开发分离，由前端人员完全控制业务逻辑，与后端的交互通过REST API实现”，鄙人简单总结了下前端MVC开发模式的一些优势：

* 前后端分离，迭代速度更快（比如以前修改一个Django View，前端页面也许需要修改，现在，只需要明确输入输出，后端API可以方便的进行迭代优化）；

* 利用一些前端MVC框架，可以实现无需再操作DOM去做展现层，而只要通过数据绑定Model即可实现（JQuery模式的Web前端开发主要的一个弊病就是需要代码来操作DOM，非常繁琐而且容易引发Bug）；

* 从`命令式开发`转到`声明式开发`（以前是通过JQuery命令式的变更DOM来展现，现在则是通过Model数据修改来显式声明，代码逻辑更干净整洁）；

* 前端模块化、组件化开发（Polymer、AngularJS等框架还带来了模块化开发的理念，前端也能实现充分的面向对象特性）；

#### Gaea 的选择

那么，有人可能会问了，即使是前端MVC，开源界也已经有如此之多的框架了，我的运维平台开发应该怎么选择呢？以Gaea为例，鄙人当时的考量是这样的：

* 针对Gaea这样的Job系统，无疑是需要前端MVC框架来实现前后端分离的；
* 每个前端MVC框架的对应学习成本如何?
* 除此之外，你选择的UI套件（运维人员很少有资源和精力去自己完成UI代码）和它的兼容性如何?

最终，鄙人选择的是[Polymer](https://www.polymer-project.org/) + [Semantic-UI](http://semantic-ui.com/)，选择它们的理由在于：

* Semantic-UI是一个新兴的非常炫酷的UI套件，虽然是借助JQuery实现的，但是封装的足够出色；

* Polymer是一个新兴的Web Component Library，它的逻辑和理念非常容易理解，上手速度快（比之Angular的确是这样的..）；

* Semantic UI和Polymer通过一些手段是可以联合使用，这也就解决了Polymer本身Element匮乏的问题；

* 鄙人做的是运维平台开发，所以浏览器兼容性完全不必考虑，Polymer的较差兼容性实际可以容忍；

具体关于这两个开源项目的细节，各位看官还是拜访它们官网比较妥当，鄙人这里就不赘述了 :)

#### 且听下回分解..

关于Gaea的一些前端选型以及理念的分享就是这些，下一篇文章可能会是关于Gaea 具体代码的一些分享和解读，敬请期待..
