---
title: "Saltstack 学习之target minions（一）"
date: 2014-05-14T20:00:00+08:00
tags: ["devops", "salt"]
categories: ["salt"]
comments: false
showMeta: false
showAction: false
---

本文总结saltstack如何定位目标主机，以及介绍一些常见使用样例。

<!--more-->

> 为什么需要定位主机？

作为配置管理软件，首先要解决的是如何确定一次推送的主机，或者说特定配置的推送目标。试问如果无法很好的确定一次或多次推送的目标，又何谈实现大批主机的分类配置管理的自动化呢？  

> saltstack的target机制？

saltstack 为此建立了一套很是完善的minion定位机制。官方术语叫“target minions”，即通过多种途径，指定minion具有的属性（比如正则匹配minion id，或是匹配minion的特定grains属性等）来区分出本次推送命令或状态的对应目标，以下是salt提供的匹配主机的方法：

- globbing （默认的匹配方式，linux shell风格的通配）
- Perl-compatible regular expressions (E，即正则匹配方式，对象同样是minion id)
- Lists （L，直接是一串minion id 列表）
- Grains （G/P，使用grains值匹配，P是使用正则匹配方式，G则是默认通配方式。）
- NodeGroup （N，master配置文件里预定义好的组别信息）
- Subnet（S，利用minion 的IP属性进行匹配）
- Range cluster （R，设置range服务器反馈值匹配?暂时没搞懂）
- Pillar（I，简单，用pillar数据匹配）
- Compound（C，组合匹配方式，以上述一种或多种方式联合匹配）

一些常见的target例子：  

	salt '*' test.ping -v
    salt 'Cloud[1-5]' test.ping -v
	salt 'test*' test.ping -v
    salt -E 'Test.*' test.ping -v
    salt -L 'TestVM01,TestVM02' test.ping -v #中间不要有空格
    salt -G 'os:CentOS' test.ping -v #同样的，不要有多余字符
    salt -N 'group1' test.ping -v
    salt -S '127.0.0.1' test.ping -v #minion只要在这个子网就会匹配
    salt -C 'E@TestVM0.* and G@os:CentOS' test.ping -v #允许逻辑运算
    #另外的，暂时没有应用过，留待以后补充……


> saltstack 源代码解读（0.17.5）

1、salt-master端
target相关代码均是零散分布于 /usr/lib/python2.6/site-packages/salt/client/\_\_init\_\_.py（1151行为pub方法），其实在master端并没有完成太多解析任务，只是完成了诸如Range server的解析等简单工作。

推送命令的原理即master 通过run_job方法统一发布job，然后该方法通过zmq发布pub，所有minion均会接收到该数据，并根据是否满足要求来完成后续工作。

2、salt-minion端
这里即完成主要的匹配工作，minion的代码位于 /usr/lib/python2.6/site-packages/salt/minion.py （主要代码为1485行开始的Matcher类，里面进一步调用了saltutil的比较方法，从而确认是否匹配），这里取其中一个简单的例子予以讲解。

    def pcre_match(self, tgt):
        '''
        Returns true if the passed pcre regex matches
        '''
        #这里注意一下，使用re模块实现正则匹配，然后反馈的
        #是boolean值，其他方法类似，即反馈minion是否match。
        return bool(re.match(tgt, self.opts['id']))


> 使用Batch选项分批次运行

saltstack 提供了-b 选项，允许对target选中的所有主机进行分批次处理，每次只推送指定的数量或百分比，直至全部推送完毕。个人感觉这个比较鸡肋，分批次运行最好还是自己定制一下，用salt的python client写个辅助脚本应该效果更好点，毕竟这个不能打断，也无法把握。

	salt '*' -b 10 test.ping #一次10台，分批推送
    salt -G 'os:RedHat' --batch-size 25% test.ping #一次跑25%
    
========================================================

参考： [salt官网](http://docs.saltstack.com/en/latest/topics/targeting/)

========================================================
