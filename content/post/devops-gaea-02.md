---
title: "运维开发探索之< Gaea.01  初探>"
date: 2015-02-01T20:00:00+08:00
tags: ["devops", "thinking"]
categories: ["devops"]
comments: false
showMeta: false
showAction: false
---

这篇文章是运维开发探索系列之Gaea篇的第一篇。

<!--more-->

#### Gaea的价值所在

可能有很多人不明白Gaea的视角以及存在的意义，鄙人在此先简单介绍下这个Gaea系列关注什么以及会分享什么：

* 首先，Gaea 的核心是基于Saltstack实现的；
* 其次，Gaea 主要关注的是运维开发中占很大比重的配置管理以及批量部署；
* 最后，Gaea 会分享它的技术理念以及实现细节，给各位分享一个实际的案例。

Ps. 鄙人实现的Gaea原型的源码会在Gaea的最后一篇文章里给出和开源。

#### 前置导读

因为`Gaea`的基石是Saltstack，所以鄙人强烈建议阅读本系列文章之前先了解一下Saltstack的基本使用。

必阅文档请见：
[Salt 官方文档](http://docs.saltstack.com)、[绿肥神的blog](http://pengyao.org)

Salt Targeting 请见：
[Salt Targeting 01](http://devopstarter.info/saltstack-xue-xi-zhi-target-minions/)、[Salt Targeting 02](http://devopstarter.info/saltstack-xue-xi-zhi-target-minionser/)

Salt Pillar & State 请见：
[Salt Pydsl Pillar](http://devopstarter.info/saltstack-ying-yong-ji-qiao-zhi-pillarde-ling-huo-guan-li/)、[Salt Pillar Scheduler](http://devopstarter.info/saltstackxue-xi-zhi-scheduler/)、[Salt State Formulas](http://devopstarter.info/saltstack-xue-xi-zhi-salt-formulas/)

Salt 高级特性和源码分析 请见：
[Salt Minion 源码解读](http://devopstarter.info/yuan-ma-jie-du-saltstackyun-xing-ji-zhi-zhi-job-runtime/)、[Salt Master-End Returner](http://devopstarter.info/saltstack-shi-yan-zhi-ce-shi-salt-event-xi-tong/)

#### Gaea 基于原生 Salt 做的改变

鄙人在使用Salt的过程中遭遇了很多的困难，以下是一些具体的典型问题所在，以及Gaea的解决方式：

* Salt的Job直到2014.7.0方才支持Ext Job Cache，而且并不是用户可控的；
  
  （因此，鄙人借助绿肥神的blog，编写了对应`salt-jobd`服务程序，实时从Zmq消息总线抓取Job的反馈数据，并归档到数据库.）
  
* Salt的LocalClient API也好，Cli也好，Salt API也好，均存在timeout不准的问题，另外，如果同步的模式下发salt-job，无法得到job_id，异步的话又没有合适的方法在合适的时刻拿到完整的反馈数据；

  （因此，鄙人编写了一个Job Engine，基于Salt的run_job原子性的Job下发函数和gather_jobinfo原子性的Job反馈收集函数实现了一个Job的整个逻辑）
  
* Salt的权限认证体系并不是很成熟 & 灵活，需要一个多维度的权限认证和授予系统来兼容各式各样的需求；

  （因此，鄙人编写了一个Authenware，做一个可插拔式的高度灵活的权限认证机制）
  
* Salt的State Job并没有一个明确的前置 or 后置动作的定义方式，require或者onfailed这样的方式鄙人并不是很接受,很多时候用户有传入自定义的检测Job完成结果或者做指定的action的需求。

  （因此，鄙人编写了一个Job Template，以做完和做好一件事情为视角，分别定义pre-action & post-action，作为一个Job集成的instance.）

* Salt原生的Job切片功能(-b参数)并不是非常好用，另外，没有办法做到一个Job下发之后的中断（然而实际情况下，比如使用Salt执行了一个错误的Job，在原生Salt支持下，便只能干看着跑完，或者手动kill_job）；

  （因此，鄙人在Job Engine里借助Py signal实现了Job的切片和中断、查询.）

当然，也许鄙人这样的实现方式并不是很合适，最恰当的方式当然还是Pull request到salt的源码，然而因为鄙人能力有限，暂时还没想到如何恰当的去和原生salt做集成，原谅我...

#### Gaea 完成的使命

Gaea在鄙人这边的应用场景会是以下这些：

* 完成运维人员的批量部署任务，直接的前端下发，简单暴力!
* 完成运维人员自定义的配置管理任务，定时保持，未授权的修改立马report!
* 完成外部数据源的汇总，结合到Target系统,实现机器灵活的角色定义!
* 完成自由的数据采集任务，为其他系统提供服务；
* etc..

其实，Gaea诞生的原因正是在于原生Salt的一些缺陷，由于鄙人没有能力把Gaea的解决方案很好的融入到Salt本身，因此它可能会是一个过渡时期的解决方案，待到Salt真正成熟 & 强大的时候，那便是回归到原生的时候了罢。

#### 且听下回分解..

Gaea的介绍到此结束，下面的文章会是一些具体技术实现和理念的分享，敬请期待..
