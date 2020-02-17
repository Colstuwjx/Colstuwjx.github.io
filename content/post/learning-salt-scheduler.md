---
title: "Saltstack 学习之Scheduler"
date: 2014-06-12T20:00:00+08:00
tags: ["devops", "salt"]
categories: ["salt"]
comments: false
showMeta: false
showAction: false
---

Saltstack原生提供schedule模块来完成计划任务的功能，基于State完成额定的定时执行的任务，本文将就此展开讲述。

<!--more-->

Salt本身提供多方面的Scheduler的配置，可以从Master 配置端、Master Pillar端、Minion配置端或Minion.d下的配置文件下配置定期计划任务。

### Pillar Scheduler

由于配置在Master Pillar端具有良好的灵活性的优势，鄙人推荐使用这种方式去完成计划任务的设定。好了，话不多说，切入正题。

Pillar允许配置Scheduler应该是0.16.x以上的版本所具备的功能，具体添加的代码如下：

	+        # Merge the pillar's schedule dict with our minion's
	+        # configuration before passing that configuration on
	+        # to the schedule.
	+        schedule_opts = {}
	+        if 'schedule' in self.opts['pillar']:
	+            schedule_opts.update(self.opts['pillar']['schedule'])
	+        if 'schedule' in self.opts:
	+            schedule_opts.update(self.opts['schedule'])
	+        opts = dict(self.opts)
	+        opts['schedule'] = schedule_opts

而Pillar Scheduler的配置其实大同小异，最终实现的效果便是Master获取到对应的配置数据然后分发到对应的Minion端。首先在/srv/pillar/top.sls中包含schedule:

	base:
  	"*"
    	- schedule
        
之后，在/srv/pillar/schedule.sls里写入具体的Schedule配置，例如：

	schedule:
  	highstate:
    	function: state.highstate
    	minutes: 10

  	testcase:
    	function: cmd.run
    	seconds: 5
    	args:
      	- 'echo 1 >> /tmp/test.cmd.log'
    	kwargs:
      	stateful: False
        
在完成上述动作后，便完成了计划任务的创建，但是想要推送到minion端使其按计划执行则还需要执行一条命令：`salt '*' saltutil.refresh_pillar`。

想要获取当前的schedule job，可以使用`salt '*' pillar.get schedule` 或 `salt '*' config.option schedule`来确定。

值得一提的是，暂时这种方式存在一点问题，那就是maxrunning默认配置为1，然后推送到minion之后只有一个minion会run schedule job，暂未解决这个问题.

### Minion.d Conf Scheduler

当然也可以直接在minion端配置schedule，方法类似，只需要在minion.d下创建一个scheduler.conf文件，里面填入同样对应的schedule信息即可。不过这种方法必须要直接的“告知minion重新载入配置”，当前已知的方式便是重启Minion service，所以存在一些运维负担。注意，这样的Scheduler配置只能通过`salt '*' config.option schedule`来检索。

========================================================

参考： 
[salt官网](http://docs.saltstack.com/en/latest/topics/jobs/index.html#scheduling-jobs)

========================================================
