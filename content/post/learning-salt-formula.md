---
title: "Saltstack 学习之Salt Formulas"
date: 2014-06-14T20:00:00+08:00
tags: ["devops", "salt"]
categories: ["salt"]
comments: false
showMeta: false
showAction: false
---

Saltstack自0.17.x版本开始引进Formulas的概念，旨在通过简化State和集成数据来实现State的友好管理，本文将就此展开探讨。更多的样例可以参见：[ Formulas.Git](https://github.com/saltstack-formulas)

<!--more-->

### 未集成的State

先来看下编写State的一般样例：

	apache:
  	pkg:
    	- installed
    	{% if grains['os_family'] == 'RedHat' %}
    	- name: httpd
    	{% elif grains['os_family'] == 'Debian' %}
    	- name: apache2
    	{% endif %}

乍看一下觉得这样写很正常，可是仔细想想吧，这样一来，apache的应用属性的判断逻辑写死在sls文件里，一旦需要更多的判断逻辑时这样的写法便稍显笨拙，那么，有什么好办法呢？

### 引进 Formulas
其实简单来说的话，Formulas即是建议使用Salt的用户编写State sls时统一使用map.jinjia来完成处理逻辑和判断，从而生成类似pillar的变量数据。

下面，鄙人同样以apache为例讲述Salt Formulas编写逻辑。Formulas结构如下：

	.
	├── /srv/salt/apache/conf.sls
	├── /srv/salt/apache/init.sls
	└── /srv/salt/apache/map.jinja

Formulas的核心便在于map.jinja的编写，如上Apache对应的map.jinja样例内容如下：

![map.jinja](/images/2014/Jun/map-jinja.png)

这样一来，apache对应的主sls（init.sls）内容则可以改写为如下：

![apache main](/images/2014/Jun/top.png)

少了复杂的判断逻辑，多了变量数据的简单使用!

### 结合Pillar的Formulas

Salt Formulas 还可以通过结合Pillar实现更多的扩展。同样以上述Apache的配置为例，需要编写conf.sls来完成apache的配置，结合Formulas Map 概念之后，它可能变成如下内容：

![apache conf map](/images/2014/Jun/extra_map.png)

注意上述配置里，除了导入map.jinja预定义内容外，还有一行source配置使用了奇怪的salt["pillar.get"]["apache:lookup:config:tmpl"]，熟悉pillar的应该不陌生，这便是结合了pillar来完成state的定义。

事先在/srv/pillar/top.sls里包含apache.sls，而后在这个pillar定义文件里添加对应内容来重写映射关系：

	apache:
 	 lookup:
        config:
          tmpl: salt://apache/files/redhat/
        server: my_custom_apache

这样一来，便可以基于Formulas map基础上做更多的特色化配置。

### Salt Formulas 制作

如你所见，Formulas其实应该算是State的集成和封装，事实上，Github也已经有很多的\*-Formulas state出现，一个标准的\*-Formulas 拥有以下结构：

	foo-formula
	|-- foo/
	|   |-- map.jinja
	|   |-- init.sls
	|   `-- bar.sls
	|-- CHANGELOG.rst
	|-- LICENSE
	|-- pillar.example
	|-- README.rst
	`-- VERSION


具体可参照Github上已有的Formulas样例，这里不再赘述。

========================================================

参考： 
[salt官网](http://docs.saltstack.com/en/latest/topics/development/conventions/formulas.html)、[Forrest Alvarez Demo](http://pan.baidu.com/s/1mgI9xiC)

========================================================
