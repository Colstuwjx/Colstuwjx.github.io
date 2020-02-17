---
title: "Saltstack 新特性测试之proxy minion"
date: 2014-07-19T20:00:00+08:00
tags: ["devops", "salt"]
categories: ["salt"]
comments: false
showMeta: false
showAction: false
---

salt 目前主要的应用场景是Linux OS下，另外还有Windows Client（Win下没用过，但是看官方issue，应该……），最近关注到官方的一个小模块提到了Proxy minion，群里也多有提及，便想着看看到底是啥存在。

<!--more-->

#### 任何设备均可被salt托管

salt proxy minion的出现，使得网管设备或者哑设备（比如sms gateway）均可被salt统一管理，而实际的管理模块或通讯接口均由用户自行编写好，具体操作内容请参见salt 官网对应的 [Proxy Minion 介绍](http://docs.saltstack.com/en/latest/topics/topology/proxyminion/index.html)


#### 开发中的新特性

首先需要注意一点，Proxy minion是2014.1.x版本引进的新特性，并且到目前为止仍处于开发阶段，仅可用作测试用途。

#### 测试环境的准备

本人的PC环境：

* vbox下ubuntu 14.04 LTS Server
* Salt master & minion 2014.1.7（配置test.ping通，不多说了）

准备好基础环境之后，需要从github下载salt官方开发人员用于测试Proxy Minion的一个小程序（Rest Server，模拟网管设备的管理接口），名字是 [salt-proxy-rest](https://github.com/cro/salt-proxy-rest)。

这个程序可能依赖两个python库，bottle和requests（其实就是web server需要的组件……），安装一下即可。

使用 `python rest.py` 运行该程序，可以将此作为一个网管设备的Web管理接口：

![alt](/images/2014/Jul/web.png)

#### 尝试运行!

至此，准备工作算是完成。在当前环境下，salt-master和salt-minion稳定运行，并且有一个提供REST接口的网管设备在独立工作，我们需要做的便是将其拉进Salt的阵营。

> 配置pillar

鄙人的minion id是docker，对应的pillar的top.sls内容配置为：
	
    root@docker:/srv/pillar# cat top.sls
	base:
  	docker:
    	- proxyminion
        
而proxyminion.sls内容则是对应网管设备的描述：

	root@docker:/srv/pillar# cat proxyminion.sls 
	proxy:
 	 rest_sample:
        proxytype: rest_sample
        url: http://127.0.0.1:8080/
        id: proxy_docker
        
这里需要注意的是，proxytype必须是在salt/proxy下已经预先定义好的，而其他的一些参数则是自己网管设备通信需要的一些数据，不一定相同。

定义好pillar数据之后，需要为之添加对应的proxy conn class和grains数据，这里鄙人使用官方sample，就偷个懒：

	root@docker:/srv/pillar# cat /usr/lib/python2.7/dist-packages/salt/proxy/rest_sample.py
	# -*- coding: utf-8 -*-
	'''
	This is a simple proxy-minion designed to connect to and communicate with
	the bottle-based web service contained in salt/tests/rest.py.

	Note this example needs the 'requests' library.
	Requests is not a hard dependency for Salt
	'''
    ……

放心，2014.1.7版本已经默认有这个sample代码。
接下来，直接test.ping试试吧！

	root@docker:/srv/pillar# salt '*' test.ping -v
	Executing job with jid 20140720110315049478
	-------------------------------------------

	docker:
    	True
	rest_sample-localhost:
    	True
        
诶，等一下，为什么多出来个key？为什么还能test.ping通？没错！这个就是ProxyMinion，而salt默认已经配置了test.ping方法兼容proxy minion了，只要写好对应的ping模块，就可以使用常规的test.ping来探测！（本例的ping代码如下）

    def ping(self):
        '''
		Is the REST server up?
		'''
        r = requests.get(self.url+'ping')
        try:
            if r.status_code == 200:
                return True
            else:
                return False
        except Exception:
            return False

rest\_sample还提供很多function，比如鄙人测试的一个service_status，修改对应的模块代码即可使之兼容proxy minion（代码路径为/usr/lib/python2.7/dist-packages/salt/modules/service.py）：

	def status(name, sig=None):
    	'''
    	Return the status for a service, 
        returns the PID or an empty string if the
        service is running or not, pass a signature
        to use to find the service via ps

    	CLI Example:

    	.. code-block:: bash

        salt '*' service.status <service name> [service signature]
    	'''
        #wjx add, denote it to work!!
    	#if 'proxyobject' in __opts__:
    	#    return __opts['proxyobject'].service_status(sig if sig else name)
    	return __salt__['status.pid'](sig if sig else name)
        
那么这时候再看看当前proxy minion管理的服务状态咋样了：

	root@docker:/srv/pillar# salt '*' service.status apache
	rest_sample-localhost:
    	----------
    	comment:
        	stopped
    	ret:
        	True
	docker:
    	False

完全和普通minion兼容！！rest_sample本身还配置了grain数据，代码位于/usr/lib/python2.7/dist-packages/salt/grains/rest\_sample.py，直接敲命令看看：

	root@docker:/srv/pillar# salt 'rest_sample-localhost' grains.items
	rest_sample-localhost:
  	  housecat: Are you kidding?
        kernel: 0.0000001
        location: In this darn virtual machine.  Let me out!
        os: RestExampleOS
        os_family: proxy

Awesome！！这样一来，一个基本的salt proxy minion就算是配置完成，Proxy Minion 的类定义代码位于/usr/lib/python2.7/dist-packages/salt/minion.py，有兴趣可以看看。

#### 可能的bug

鄙人在本机测试时，Minion Docker在尝试fork出一个ProxyMinion过程中间报错，说\_running参数没有配置，在添加代码后通过（即位于minion.py代码里）

	class ProxyMinion(Minion):
    '''
    This class instantiates a 'proxy' minion--a minion that does not manipulate
    the host it runs on, but instead manipulates a device that cannot run a minion.
    '''
    def __init__(self, opts, timeout=60, safe=True):  # pylint: disable=W0231
        '''
        Pass in the options dict
        '''
        #wjx add, maybe a bug
        self._running = None

        # Warn if ZMQ < 3.2
        if HAS_ZMQ and (not(hasattr(zmq, 'zmq_version_info')) or
                        zmq.zmq_version_info() < (3, 2)):
		……
        
#### 聊聊Proxy Minion

Proxy minion使得salt针对网管设备的配置管理成为可能，不过想要实现一个ProxyType的ProxyMinion的完全管理，可能需要编写很多额外的module去支持它的运行。

在大公司复杂的网络环境下，完全可以针对此编写对应SNMP管理模块或者针对OVS编写对应的管理模块，尔后通过salt统一托管，毕竟Salt有一套完善的配置管理体系啊！
