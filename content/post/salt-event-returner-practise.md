---
title: "Saltstack 实战之event + returner的灵活使用"
date: 2014-05-15T20:00:00+08:00
tags: ["devops", "salt"]
categories: ["salt"]
comments: false
showMeta: false
showAction: false
---

最近看到绿肥大神的那篇salt event系统的介绍，觉得可以通过这种方式灵活的建立起一个反馈机制，主要用到的是绿肥大神使用的event监听以及returner本地返回。

<!--more-->

（Ps. 关于这两块具体的内容和运作机制，鄙人将在之后的文章介绍。）

> 利用event + returner建立 job 审计机制

这里主要参考的就是绿肥大神的文章，由于使用的是已建立的event system，只需要在master端部署，而这无疑是极为方便的。

（ 试想一下吧，如果单纯使用returner，比如mysql，那minion端便需要已安装mysql-python接口模块，对于环境的部署又添了一层需求。简单就是美啊，越简单的部署方案越是实用! ）

基于 salt 的 event system 写入mysql job 数据的具体实现细节请参见绿肥大神的博客 [基于Event建立Returner](http://pengyao.org/saltstack_master_retuner_over_event_system.html)，没啥说的，牛！下面是主要的实现脚本    salt\_event\_to\_mysql.py

	# salt_event_to_mysql.py
    # written by PengYao.org
	#!/bin/env python
	#coding=utf8

	# Import python libs
	import json

	# Import salt modules
	import salt.config
	import salt.utils.event

	# Import third party libs
	import MySQLdb

	__opts__ = salt.config.client_config('/etc/salt/master')

	# Create MySQL connect
	conn = MySQLdb.connect(host=__opts__['mysql.host'], 	user=__opts__['mysql.user'], 	
    passwd=__opts__['mysql.pass'], 
    db=__opts__['mysql.db'], 
    port=__opts__['mysql.port'])
	cursor = conn.cursor()

	# Listen Salt Master Event System
	event = salt.utils.event.MasterEvent(__opts__['sock_dir'])
	for eachevent in event.iter_events(full=True):
    	ret = eachevent['data']
    	if "salt/job/" in eachevent['tag']:
        	# Return Event
        	if ret.has_key('id') and ret.has_key('return'):
            	# Igonre saltutil.find_job event
            	if ret['fun'] == "saltutil.find_job":
                	continue

            	sql = '''INSERT INTO `salt_returns`
                		(`fun`, `jid`, `return`, `id`, 								
                        `success`, `full_ret` ) 
                        VALUES (%s, %s, %s, %s, %s, %s)'''
            	cursor.execute(sql, (ret['fun'], ret['jid'],
                			   json.dumps(ret['return']), ret['id'],
                               ret['success'], json.dumps(ret)))
            	cursor.execute("COMMIT")
    	# Other Event
    	else:
        	pass

不过，这里还想追加点个人见解：

> * 作为配置管理系统，salt master\minion两端的执行情况都应很好的把握。

> * 尽量减少minion端的主动抓取，统一方向的配置和部署会更容易维护，还是那句话，简单就是美!

salt 分为两方面的推送，一方面 master 会主动的推送部署到 minion 端，另一方面minion也可以主动从 master 抓取配置（通过 salt-call 或 scheduler ），本文的主要目的便是实现一个两端的全局把握，确保相应的执行结果都落地到对应目标。

但是，通过执行绿肥大神的 `python salt_event_to_mysql.py` ，仅仅是可以将 master 端触发的所有job event都存档到对应的 mysql returner 中。在salt 0.17.5版本及以下，鄙人很确定，minion 端没有 反馈到master的returner([saltstack issue#10500](https://github.com/saltstack/salt/commit/537f96fe14ad515c8cd16f7c1962a625a461ae98))。

那minion端呢？比如salt-call执行的命令，写入到minion的schedule? 答案是都拿不到，那这可如何是好？且请接着看下去。

其实，我们可以针对minion端执行情况单独使用returner落地（存档到本地日志文件）。比如配置了salt schedule，允许minion每隔60s主动从master拉取一次highstate状态。

		schedule:
  		  highstate:
    	        function: state.highstate
    			minutes: 1
    			returner: local_return

那么，接下来需要配置一下/srv/salt/\_returners/local\_return.py:

	#local_return.py
	#coding:utf-8

	def __virtual__():
    	return "local_return"
        
	def returner(ret):
    	f = open('/var/log/salt/local_returner.log', 'a+')
    	f.write(str(ret)[1:-1] + "\n")
    	f.close()

因为schedule已配置数据流转到local\_return，所以执行下sync即可：`salt '*' saltutil.sync_returners`。
这样一来，便可以收集到minion端highstate job结果信息：

	tail /var/log/salt/local_returner.log
    
    'fun': 'state.highstate', 
    'return': {.{..  'name': 'echo yes >> /tmp/tmp.log',  'result': True..}}, 
    'jid': '20140515094711022582', 'pid': 8617, 'id': 'TestVM'

美中不足的是，如果需要收集salt-call的job执行结果，那么便需要主动的通过指定returner实现，比如`salt-call state.highstate --return local_return`

如果想修改默认returner，额，暂时没找到，修改源代码（ /usr/lib/python2.6/site-packages/salt/cli/caller.py， 131行 ）倒是可以，不过太麻烦了……

至此，salt master与minion两端的job执行结果的收集算是完成了，比如想查询某天salt-minion schedule highstate job的执行情况或者salt-master端某个job（指定jobid）的执行结果，都可以查询到了！

由于上述所讲的是基于python代码开发建立的，所以可以灵活的添加额外的功能，比如事件审计（在有出错时发邮件啥的）。

========================================================

参考：   
[salt官网](http://docs.saltstack.com/en/latest/topics/event/index.html)、
[绿肥神的blog](http://pengyao.org/saltstack_master_retuner_over_event_system.html)

========================================================