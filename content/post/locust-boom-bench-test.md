---
title: "使用locust + boomer实现对非web组件的压测"
date: 2018-01-20T20:00:00+08:00
tags: ["benchmark", "devops", "test"]
categories: ["test"]
comments: false
showMeta: false
showAction: false
---

最近有在编写类似SkyDNS的面向服务间调用的智能DNS服务，开发完成后如何对DNS服务本身进行压测有点被难倒了。市面上对HTTP类请求的压测测试很多，然而对非Web组件的压测工具则非常少见...

<!--more-->

#### 前言

在努力翻阅互联网上的开源项目后，终于找到了这样一款压测工具[locust](https://github.com/locustio/locust)，它是用Python实现的一款可以自己编写task进行测试的压测框架，并且提供友好的UI。本文即是简单总结一下体验该款压测工具的过程，BTW，笔者对测试没有什么经验，如有谬误，欢迎互联网上的大佬们一起交流、指教。

#### 架构设计

下面，我们不妨来简单看看locust的架构。

`locust`本身设计的非常简单轻量，经典的压测架构 master -> slave的形式，master只负责数据的收集和消息的广播等操作，task的执行均由slave执行。

分发task的话需要编写对应的[locustfile](https://docs.locust.io/en/latest/writing-a-locustfile.html)，文件内容里即指定task和相应的执行数量、权重、限制并发数等。locust原生的slave是通过gevent来实现高并发的测试请求，而这里我们也可以使用[boomer](https://github.com/myzhan/boomer)，
这是一款Github上开源的Golang版实现的slave，完全兼容locust的协议！

值得一提的是，这位开源boomer的大佬貌似是服务端测试领域的专家，里面一些提供的用例也非常值得挖掘，比如针对UDP服务实施的[udptest](https://github.com/myzhan/boomer/blob/b669e7600ff989ccbafaa434641efce69ac52235/examples/udp_perf_proxy.go#L12)，完成对UDP请求的copy和分析上报。

#### dns测试

马不停蹄，下面我们赶紧编写一个dns的测试用例，看看效果！[boomer](https://github.com/myzhan/boomer)的文档还是很清晰的，直接套上官方的样例，实现一个`test_dns`:

```
...

func test_dns() {
	/*
		一个简单的 DNS查询操作
		TODO: complicated dns query using https://github.com/miekg/exdns/blob/master/q/q.go
	*/
	startTime := now()
	addrs, err := net.LookupHost("www.baidu.com")
	endTime := now()

	log.Println(float64(endTime - startTime))
	if err != nil {
		boomer.Events.Publish("request_failure", "dns", "udp", 0.0, err.Error())
	} else {
		boomer.Events.Publish("request_success", "dns", "udp", float64(endTime-startTime), int64(len(addrs)))
	}
}

...
```

上面这个例子即是利用本地的dns解析`www.baidu.com`，如果解析成功则上报一次success，否则即上报一次failure。这里的上报机制即是通过locust本身提供的event bus，slave可以将自己的测试结果上报到服务端，也可以广播一些message给master，比如声明slave节点的在线离线等。

我们不妨通过本地执行该task验证下测试用例是否可行，这里笔者编写了一个[`boomer.go`](https://github.com/Colstuwjx/golang-examples/tree/master/src/test/boomer)，可以通过如下命令测试dns的用例：

```
# go build -o a.out boomer.go
# ./a.out --run-tasks dns

2018/01/20 14:16:39 Running dns
2018/01/20 14:16:39 resolving www.baidu.com.. cost time:  2
```

#### locust+boomer进行压测

实际用locust做dns服务的压测也比较简单，首先安装并启动locust master（这里dummy.py即占位的fake脚本，我们用boomer来充当实际的task声明和执行者）：

```
# pip install locust
# locust -f dummy.py --master --master-bind-host=127.0.0.1 --master-bind-port=5557
```

然后启动boomer充当slave，并连接locust master：

```
go run boomer.go --master-host=127.0.0.1 --master-port=5557
```

如此一来，我们便可以在locust的管理UI上下发压测任务：

![locust](/images/2018/Jan/locust.jpg)

Em.. 简单在笔者的mac笔记本测试效果还不是很理想... 可能和机器本身性能也有关系：

![result](/images/2018/Jan/result.jpg)

Em.. DNS的测试维度还比较单一，测试slave也需要分布在不同机器和网络单元，量级也需要根据请求处理的增长和处理趋势判断出瓶颈，下周搞搞..

就酱！

#### 参考

http://myzhan.github.io/2016/03/01/write-a-load-testing-tool-in-golang/

https://github.com/myzhan/boomer

https://github.com/locustio/locust
