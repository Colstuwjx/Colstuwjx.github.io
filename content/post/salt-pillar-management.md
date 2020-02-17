---
title: "Saltstack 应用技巧之Pillar的灵活管理"
date: 2014-06-14T20:00:00+08:00
tags: ["devops", "salt"]
categories: ["salt"]
comments: false
showMeta: false
showAction: false
---

Saltstack使用Pillar和Grains数据来管理和分类Minion，其中Grains数据作为Minion的“固有属性”存储在Minion端，而Pillar则作为“变量数据”存储在Salt-Master端，本文就如何灵活的应用和管理Pillar来实现分类与配置部署的话题予以探讨。

<!--more-->

### 常见的Pillar配置
一般来说，常见的配置Pillar样例如下所示：
 
 	.
    ├── top.sls
	├── appcase
	│   ├── conf.sls
	│   └── init.sls
	├── basiccase
	│   └── init.sls
	├── schedule.sls
	└── users.sls

通过定义top.sls来统一的进行管理，而从分层的sls结构很好的定义一些应用或者OS基础层的相关Pillar参数。

然而，由于Saltstack本身提供的Pillar暂时还不够强大，比如像灵活控制Top.sls或者使用Extends这样的特性在原生的支持上便很难达成，网上因此也出现了很多介绍如何结合Cobbler或者Reclass来辅助增强Pillar的文章（都是非常优秀的技术博客文章，鄙人均一一拜读，感谢大神们的分享）。

鄙人这里不打算介绍如何结合第三方的应用来扩展Pillar，而考虑使用Saltstack本身的良好扩展性来达成效果，具体细节且待我一一说来。

###使用Python扩展的Pillar
其实真的应该说是一叶障目，为什么不能简单的直接使用Python去定制Pillar呢？通过查看Saltstack源代码（这便是开源的魅力所在！），我发现其使用类似 `yaml_load()` 这样的方法来解析Pillar sls（应该state sls也是一样，这里没测试过~）。因此，可以通过将top.sls转换成如下格式来实现灵活的管理：

	#!py
	#base:
	#  '*':
	#    - schedule
	def pillar_helper():
    	return {'base': {'*': ['schedule'], 'TestVM': ['users']}}

	def run():
    	return pillar_helper()
        
这样一来，通过pydsl的方式去管理，无论是自定义额外模块还是编写额外方法去判断处理都是可行的了。为了更好的去理解，简单介绍一个应用的实例吧，结构如下所示（注意与常见配置的一些差异~）：
 
 	.
	├── appcase
	│   ├── conf.sls
	│   └── init.sls
	├── basiccase
	│   └── init.sls
	├── nodes
	│   └── TestVM.sls
	├── schedule.sls
	├── top.sls
	├── top.slsc
	└── users.sls
    
其中，top.sls实际算是pydsl格式文件，内容为：
	
    #!py
	#base:
	#  '*':
	#    - schedule
	def pillar_helper():
    	return {'base': {'*': ['schedule'], 'TestVM': ['users', 'nodes.TestVM']}}

	def run():
    	return pillar_helper()


而这样一来，便可以通过Python脚本的扩展方式灵活的管理每台具体主机或者组的Pillar参数，如上的TestVM.sls即定义了TestVM自身的Pillar值：

	[root@monitor pillar]# cat /srv/pillar/nodes/TestVM.sls 
	role: blog_app

而使用 `salt '*' saltutil.refresh_pillar` 刷新 pillar 数据后可以确认下，的确是预想的那样：

	[root@monitor ~]# salt 'TestVM' pillar.item role
	TestVM:
    	----------
    	role:
        	blog_app

实际还可以根据需求将此扩展到适合的结构，毕竟可以使用Python嘛，扩展性无与伦比。