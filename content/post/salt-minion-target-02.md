---
title: "Saltstack 学习之target minions（二）"
date: 2014-05-14T20:00:00+08:00
tags: ["devops", "salt"]
categories: ["salt"]
comments: false
showMeta: false
showAction: false
---

本文紧接上一篇，就target的各分类方式的详细用法予以讲解。

<!--more-->

> Grains

首先一点，需要注意的是，minion的grains信息在minion启动时便会生成和加载，之后便以静态数据的形式存在。

Grains的匹配在前文已经有所提及，实现原理便是读取grains的dict数据，而后与tgt字串进行匹配，当然，它支持嵌套key-value形式，如：

	salt -G 'ec2_tags:environment:*production*' test.ping -v

上述命令即寻找grains的ec2_tags grain数据对应的environment key，且value包含production的所有minion。

系统默认收集grains数据的源码可以参见[salt grains core.py](http://github.com/saltstack/salt/blob/develop/salt/grains/core.py)

由于salt默认提供的静态grains数据覆盖面不够，为此我们可以做一些自定义grains的工作，从而达成业务上的需求。

- /etc/salt/grains或/etc/salt/minion.d/xx.conf (在minion端建立grains)

  	  roles:
          - webserver
          - memcache
  	  deployment: datacenter4  
  	  cabinet: 13  
        cab_u: 14-15
- /srv/salt/\_grains/ (在master端分发到各minion)

  	  # collect.py
        #!py
        #返回的应该是一个dict，key-value形式
        #之后通过salt '*' saltutil.sync_grains分发
        def run():
            return {"test_data": "yeah"}
        run()

这里注意一下，上面所述的两种grains自定义其实是有各自的应用场景，前者可以是特定的少数minion配置独立的grains数据，以方便target而建立的，比如临时的打上一个集群的标签，以便快速定位和部署，其部署位置在于 **minion端** 。而下面的\_grains下放置的py脚本则是完全的自定义grains，为的是统一的添加额外的grains数据，比如一些业务区分信息等，于 **master端** 部署，尔后分发到各个minion。
      
> 分组

前文提到可以定义Node Group，将minion有机的区分组别，比如电信机房的归为一个组，联通机房归为另外一个组（怎么实现? 可以考虑用自定义grains!）

官方文档中介绍的Node group使用方法实在有点坑，必须要写入到master的配置文件才可以生效（还需要重启salt-master!!）。不过，在看了绿肥大神的博客后豁然开朗，“有大牛就是好啊! ”。

实际上，可以通过在/etc/salt/master.d/nodegroups.conf文件（没有就新建）写入相关内容，相应的，默认salt-master可以识别出对应定义的组别信息！
下面是一个简单的 nodegroups.conf 样例：

    nodegroups:
  	  test1: 'E@TestVM.*'  
  	  test2: 'L@salt01,salt02'  
  	  test: 'N@test1 or N@test2’  

尝试一下 `salt -N test test.ping -v` 吧！ 你会得到想要的答案! 

对于分组，个人认为，应该在充分定义好各minion相应信息（最重要的应该是grains数据）之后，再写入到nodegroups.conf文件里。分组大可不拘泥于数量和差异，以业务、服务器类别、OS等多维度划分，这样将会极大便利于后面的部署和维护！

========================================================

参考：   
[salt官网](http://docs.saltstack.com/en/latest/topics/targeting/grains.html)、
[绿肥神的blog](http://pengyao.org/salt-nodegroup-complex.html)

========================================================
