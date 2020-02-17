---
title: "建立Saltstack运转机制之个人做法（一）"
date: 2014-06-11T20:00:00+08:00
tags: ["devops", "salt", "best practice"]
categories: ["salt"]
comments: false
showMeta: false
showAction: false
---

一个企业的IT系统的运转机制是非常复杂的，如何结合Saltstack加入自动化运维的基因，不同的运维工程师都有各自独到的见解和做法，本文就此介绍下鄙人参考大神们做法得出的一些拙见。

<!--more-->

## Server视角的自动化

每个对外提供服务的最小实体可以即是“服务器”，一台台定义好角色、属性的服务器或独立、或集群、或分布式的对外提供各式各样的服务。

服务器存在多种状态，简单的可以分为：上线、工作中、下线。

### 上线

首先是上线动作， 这应该是最先需要完成的也应该是最容易实现的自动化过程。上线机器需要完成资源的配置、CMDB数据的维护、OS的安装和软件安装以及后面的相关业务的部署。

上线过程应该在用户提出需求后立即启动自动化进程，以Job的形式一次性运行完成，结合Saltstack的Pillar及Grains完成服务器自身数据属性的定义和配置（之后再贴样例）。

同时，为所有基础层的服务编写对应的sls，比如SSH、Cron、Yum等等，配备一个基准配置，在完成pillar和grains的server数据生成后，后台自动的开始同步过程，从Master端抓取标准配置，配置和推送到对应新上线的服务器。

仅仅完成基础层面的部署还没有完成服务器上线的全部任务，针对上线的服务器还需要定义其组别信息（Nodegroup），以及制定额外的、针对业务层面的sls同步状态。比如新上线一个Web APP工具平台用途的服务器，那么相应需要保证Apache服务的可用性以及相关配置文件的准确性，这便需要额外的动作去定义。

### 工作中

工作中的机器需要完成一系列持久化和事件处理的动作，而这些均是可以通过自动化来避免重复劳动。

首先是持久化服务，也即是保证一些指定服务的常驻运行，一旦其stop或异常时主动的进行service restart或其他动作，可以配置成salt schedule定时的执行和同步。下面是keep-running.sls示例：

    	include:
  		- testcase.keep-running
  		- testcase2.keep-running
		test_cmd:
  		  cmd.run:
    	    - name: "whoami"


然后再在对应salt-master端的/etc/salt/master.d/scheduler.conf或对应pillar写入相应的schedule即可。

另外，服务器在同步状态或部署时可能出现异常，这可以结合Salt的Reactor机制作出响应，在抓取到异常tag事件后作出预先定义好的动作。

### 下线

针对服务器的下线动作，首先需要清除pillar数据，并清理掉对应的Nodegroup定义内容。其次，需要完成CMDB数据的更新，确保服务器存在痕迹被很好的清理。
