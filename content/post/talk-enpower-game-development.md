---
title: "如何为游戏开发赋能的一些思考"
date: 2020-08-23T15:00:00+08:00
tags: ["talk", "game-development", "enpower"]
categories: ["talk"]
comments: false
showMeta: false
showAction: false
aliases:
    - /talk-enpowering-game-development/
---

最近一直在思考怎么能为游戏业务研发赋能，让他们能够更好地专注于游戏本身精品化内容的打造，这里权且记录一下一些想法。

<!--more-->

依笔者个人的见解，围绕**为游戏业务赋能**这个核心目标的话，可以做的事情主要有这几块（这里我们只聚焦在网络游戏的后端部分）：

- 游戏服务端开发框架

- 消息通信通用组件

- 接入层和负载均衡

- 公共基础服务

- 大数据分析平台

- 游戏管理平台

下面我选几个目前个人比较关注的部分讲讲。

### 游戏服务端开发框架

笔者调研了一下业内一些游戏开发相关的框架：

- 网易于 2012 年左右开源的使用 nodejs 编写的 [pomelo 框架](https://github.com/NetEase/pomelo)

- [xtac](https://github.com/xtaci) 于 2015 年左右开源的使用 Golang 编写的[gonet2 游戏开发框架](https://github.com/gonet2)

- 业内闻名的[云风大佬](https://github.com/cloudwu)于 2012 年开源的使用 C 编写的 [skynet](https://github.com/cloudwu/skynet) 框架

*待补充各个框架优劣比较*

不过比较可惜的是，除了云风大佬还在持续维护他开源的 skynet 项目以外，其余两个项目均已陷入停止开发状态。

### 接入层和负载均衡

这阵子因为工作原因接触和了解了 4 款网络游戏背后的后端架构，笔者惊奇地发现它们大都使用了独自开发一个所谓的 gateway/gate 节点服务来和客户端建立 TCP 长连接通信，而不是像经典的互联网架构那样，遵循 DNS -> GSLB/LB/FE -> service 这样的接入层架构。

这仔细想想也是有原因的，网络游戏一般追求的是一个沉浸式的用户体验，因此大多数都是 socket 式的网络通信方式，那么为了能够做到相对地可以横向扩展（scale out），它就必须抽象出一个所谓的 gateway 服务来分发玩家请求。与互联网服务不同的是，它有时候并不是一个复杂的链式调用请求，而更像是一个基于节点角色和消息的多点通信的结构，也因此有些游戏会设置一个 World 节点服务来统筹玩家调度和各种角色节点之间的协同。

笔者也了解了一下，腾讯互娱是 TGW + [tconnd](https://sdk.gcloud.tencent.com/documents/details/%E7%BD%91%E7%BB%9C%E8%BF%9E%E6%8E%A5%20Connector) 这样的接入层架构，并搭配 [tbus](https://github.com/hellokangning/TSF4G) 来和背后的 GameServer 做通信。

*待补充具体分析和实现思路*

### 游戏管理平台

除了上面两块，笔者个人认为游戏管理平台也值得重点关注。这主要分为运维面和运营面两个维度：

- 运维面重点解决区服数据的建模和区服资源的治理，以及服务管理相关的需求（包括版本升级、开关服等）

- 运营面则关注通过工具形式解决运营团队的需求，例如封禁玩家、发布公告、爆率设置等等

*待补充案例和实现思路*

### 结语

这里目前只是笔者个人的一些琐碎想法的记录，后面希望能够不断践行然后充实相关的内容吧！
