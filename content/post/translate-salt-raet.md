---
title: "[ 翻译 ] salt 新通信架构——salt raet（Github篇）"
date: 2014-07-06T20:00:00+08:00
tags: ["devops", "salt", "translate"]
categories: ["salt"]
comments: false
showMeta: false
showAction: false
aliases:
    - /-fan-yi-salt-xin-tong-xin-jia-gou-salt-raet/
---

Saltstack官方在salt 2014 介绍视频中引入了salt raet概念，salt raet是继Salt-Zeromq, Salt-Ssh之后的第三套通信体系，全名为Reliable Asynchronous Event Transport,即基于事件的可靠异步传输协议 —— by 译者

<!--more-->

原文地址：https://github.com/saltstack/raet

### 正文

> 为什么要研发Salt Raet?

现代大规模的分布式应用架构，其组件均是分布在互联网上的多个主机和多个CPU内核，往往是基于一个消息或事件总线，允许不同的分布式组件之间相互异步通信。通常情况下，消息总线是某种形式的消息队列服务，如AMQP或ZeroMQ。消息总线支持通常被称为发布\订阅模式的信息交互方式。

一个具备完整功能的消息队列服务拥有很多的优点，然而，其存在的缺陷之一便是大规模应用环境下的性能问题。

一个消息队列服务完成两个独立而互补的功能。

* 第一个是在互联网上消息的异步传输。

* 第二个是消息队列的管理，这便是通过队列实现的诸如消息的识别、跟踪、存储，以及发布者和订阅者相互之间的消息分发。

对于众多应用程序而言，一个消息队列服务的优势之一便是通过API能很好的隐藏服务背后的细节，也就是其来自各个客户端的消息队列管理的复杂性。

但是，消息队列服务存在的主要缺陷便是大规模应用环境下的扩展性，其消息的容量、消息的定时以及内存、网络和CPU处理能力相关的需求往往成为关键，而客户端对于服务性能的调整往往显得无能为力。MQ服务在分布式的应用环境下通常会成为瓶颈所在，而更复杂的MQ服务，如AMQP，则会在高负载下变得不可靠。

异步事件的网络传输和消息队列管理之间功能的分离使得每个功能模块可以独立的调整各自大规模环境下的性能。

绝大部分的MQ服务是基于TCP/IP网络传输协议。TCP/IP对于网络通信有显著的延迟效应，因而不太适合用于分布式事件驱动式应用程序交互的异步特性。这其中主要的原因便是在于TCP/IP为了支持流传输针对连接的建立和连接的关闭以及失败连接的处理。从根本上来说，TCP/IP是基于大规模的连续数据流作出了诸多优化，而不太适用于大量小规模异步事件或消息的传输。其在小规模应用系统下不成问题，而一旦达到一定规模，相关的通信特征的差异问题将会凸显。

UDP/IP 低延迟和无连接的特性注定它更适用于许多小规模异步消息的传输。UDP/IP本身的缺点在于它是不可靠的传输协议。

这里所需要的便是一个适配的针对UDP/IP协议增添可靠性而同时无损其低延迟及扩展性的传输协议。

一个事务型协议，比流协议更适用于为异步事件提供可靠传输。

进一步来说，由于大部分的MQ服务是基于TCP/IP协议，他们也更倾向于使用HTTP或者保证安全通信的TLS/SSL。虽然使用HTTP能够轻松的提供基于Web的集成系统，但是长远来说它也会成为高性能系统的瓶颈所在，TLS对于一个安全系统而言，其性能和漏洞两方面也同样存在问题。

椭圆曲线加密，另一方面来说，在相对于其他实现方法更低的性能需求的前提下增强了系统的安全性。LibSodium提供了一个开源的椭圆曲线加密库，用于验证和加密支持。CurveCP协议基于LibSodium提供了一个引导安全网络信息交互的握手协议。

最后，在分布式并发事件驱动的应用环境下管理和协调处理器资源（CPU、内存、网络）的一个最佳途径便是使用一种称为微线程的东西。一个微线程应该说是一个程序语言级的特性，它在不比函数调用消耗更多资源的情况下实现了代码逻辑上的并发。

微线程使用协同工作的多任务处理来取代线程和/或进程，其避免了许多诸如资源竞争，上下文切换，和进程间通信的复杂性，同时提供了更高的性能。

由于所有协同工作的微线程均运行于一个进程，也就造成一个简单的微线程应用的资源调用被限制在一个CPU核心。为了使得所有的CPU核心均得到充分的利用，应用程序需要能够为每个CPU核心运行至少一个进程。这就需要同一台主机的进程间通信。但是不同于传统的多进程处理方式，即一个进程完成一个逻辑并发的功能，一个基于微线程的多进程程序不再使用一个微线程处理一个逻辑并发功能的模式，而微线程的总数是取决于总的进程的最小数目的限制，并且其不超过CPU的核心数量。这将优化CPU的处理能力，同时最大限度地减少进程上下文切换的开销。

一个使用这样的微线程-多进程架构平台的典型例子便是Erlang。事实上，Erlang模式的成功为RAET方案的可行性提供了强有力的支持。
进一步来说，一个潜在的问题可能是：我们为什么不使用Erlang呢？
很不幸的是，Erlang生态系统跟Python相比较而言有些时候显得有些局限性，而它本身则使用了一种不太友好的语法结构。RAET的设计实现目标之一便是充分集成现有的Python生态系统丰富的类库和知识集，同时又能够方便的开发一个基于微线程-多进程架构模型的分布式应用程序。我们的终极目标便是想两全其美。

RAET设计用于借助微线程-多进程应用框架提供在互联网上安全可靠的、可扩展的异步消息\事件传输，其使用UDP协议来完成主机间的通信，LibSodium来完成认证、加密，CurveCP握手协议来达成安全引导。

相应的队列管理和微线程应用程序的支持由Ioflo提供。RAET可以算作是Ioflo的一个互补项目，它使得多个Ioflo程序可以通过网络结合在一起，作为一个分布式程序的一部分协同工作。

导致开发RAET的一个主要驱动因素便是使得Saltstack具备更好的扩展性的需求。Saltstack是一个使用Python编写的远程执行和配置管理平台。Saltstack使用Zeromq作为它的消息总线或者说消息队列服务。Zeromq是基于TCP/IP传输协议实现的，因而也存在上述相应的TCP/IP基础架构下的延迟及非同步性问题。此外，因为Zeromq是通过一个特殊的“套接字”将队列管理和传输集成在一起，其大规模应用环境下的队列的独立传输性能也成为问题所在，甚至于跟踪Bug也存在一定的困难。

### 安装

当前的RAET提供Pypi安装方式，在类Unix系统下通过pip指定如下命令来完成安装：

`# pip install raet`

### 介绍

当前的Raet支持两种通信方式：

* 主机间通过UDP\IP协议套接字通信;

* 通过Unix域（UXD）套接字实现的同一主机进程间通信。

一个基于Raet的应用程序架构如下图所示：

![Salt Raet程序架构](/images/2014/Jul/RaetMetaphor.png)

### 组件的隐喻命名

下列组件的形象命名仅仅是为了保持一致性而设计，与Ioflo并不冲突。

#### Road，Estates，Main Estate

* UPD通道便是一个“Road”;

* 一个Road的成员便是“Estates”（就像现实中房屋前方的道路）;

* 每个Estate拥有一个唯一的UDP主机端口地址“ha”，一个唯一的字符串“name”和一个唯一的数字ID“eid”;

* 一个Road上的Estate便称为Main Estate;
 
* Main Estate允许其他Estates通过Join方法（key交换）加入到Road上并且允许（CurveCP）相互之间的信息交易;
 
* Main Estate同样负责担任其他Estates的消息路由。

#### Lane，Yards，Main Yard

* 每个Estate都可以是一个“Lane”，这便是一个UXD通道;

* 一个Lane的成员便是“Yard”（简单来说便是一个Estate的细分点）;

* 一个Lane的每个Yard成员都有一个唯一的UXD文件名称、主机地址‘ha’和一个唯一的字符串“name”。这个类通常也有一个Yard的数字ID“yid”用于生成Yard name但是它不是Yard实例的属性之一;

* Lane name和Yard name结合起来便可以形成一个唯一的文件名，它是UXD的主机地址‘ha’;

* 一个Lane上的Yard便是一个Main Yard;

* Main Yard负责形成Lane并且允许其他Yards加入到该lane;在此之前这还没有一个正式的处理过程。当前这会设置一个由Main yard维护的标志，其将会把任何事先未在Yards列表里的Yard发送过来的数据包Drop掉。另外，文件权限的设置可以用来阻止伪Yards与Main Yard交互。

* Main Yard还负责其他Lane上的Yards之间的消息路由。

### 运行IoFlo

* 每个Estate UDP接口运行于一个在单个IoFlo House上下文中工作的RoadStack（UDP套接字）（因此可以把运行UDP Stack的House看作是Estate组成的庄园别墅）;

* 每个Yard UXD接口运行于一个在单个IoFlo House上下文中工作的 LaneStack（Unix域套接字）（因此可以把运行UXD Stack的House看作是Estate的附属Houses、Tents或Shacks）;

* “庄园”House的特殊之处在于其可以运行在针对Estate的UDP Stack和针对Main Yard的UXD Stack两套环境下;

* 运行Main Estate UDP Stack的House可以被认为是Mayor's House;

* 在上下文中，一个House便是一个数据存储。共享的数据存储可以被以一个点号分隔路径的唯一共享名称定位。

### 路由

鉴于上面所描述的Ioflo运行架构，其依照如下方式完成路由：

* 为了定位一个特定的Estate，Estate name需要相应的指定;
* 为了定位一个Estate下的特定Yard，Yard name需要相应的指定;
* 为了定位一个House下的特定Queue，Share name需要相应的指定。

UDP stack 将Estate name映射到UDP主机地址和Estate ID，而UXD stack 将Yard name映射到UXD主机地址，任意IoFlo行为的存储将Share name映射到共享引用。

因此，路由是从：在一个源Share、源Yard或是一个源Estate中的队列定义好的一个源节点到：一个源Share、源Yard或是一个源Estate中的队列定义好的目的节点。

这里需要两个三选一的节点，一个是源节点，另一个则是目标节点

源节点（Estate name，Yard name，Share name）。

目标节点（Estate name，Yard name，Share name）。

如果三合一中的任一元素均是None或者空，那么便会使用默认参数值。如下便是一个路由本身信息的消息正文样例。

    estate = 'minion1'
    stack0 = stacking.StackUxd(name='lord', lanename='cherry', yid=0)
    stack1 = stacking.StackUxd(name='serf', lanename='cherry', yid=1)
    yard = yarding.Yard( name=stack0.yard.name, prefix='cherry')
    stack1.addRemoteYard(yard)

    src = (estate, stack1.yard.name, None)
    dst = (estate, stack0.yard.name, None)
    route = odict(src=src, dst=dst)
    msg = odict(route=route, stuff="Serf to my lord. Feed me!")
    stack1.transmit(msg=msg)

    timer = Timer(duration=0.5)
    timer.restart()
    while not timer.expired:
        stack0.serviceAll()
        stack1.serviceAll()


    lord Received Message
    {
        'route':
        {
            'src': ['minion1', 'yard1', None],
            'dst': ['minion1', 'yard0', None]
        },
        'stuff': 'Serf to my lord. Feed me!'
    }

### UDP/IP Raet协议的具体细节

UDP Raet协议便是基于一个预编码的隐喻命名约定，换句话说，就是estates 关联到一个Road。核心的对象便是由如下包提供：raet.road

#### Road Raet 生产环境UDP/IP端口

Manor Estate端口为4505，而其他Estates端口为4510

#### 数据包格式

Raet使用一个带有数个字段的排序好的字典数据来初始化一个数据包的数据，而其实大多数的字段都共享在下面头部数据格式，因此仅仅有少量唯一的字段会显示在这里。

#### 唯一的数据包字段
	sh: 源主机IP地址 (ipv4)
	sp: 源主机IP端口
	dh: 目的主机IP地址(ipv4)
	dp: 目的主机IP端口
    
#### 数据包的包头格式

数据包头部的.data部分是一个排序好的字典数据，其常常用于创建一个准备传送的数据包或是收取一个数据包的相应字段。什么字段应该被包含到数据包头依赖于数据包头的类型。

#### Header的编码

当前Raet支持三种类型的头部编码格式。

	RAET 本地格式。
    这便是一个在可读性和大小方面做过权衡和优化的精简的ASCII文本格式。该模式作为RAET通信的默认选项。

    JSON 数据格式。
    这是一个最可视化的格式而且具备一些兼容性上的优势。

    二进制 数据格式。
    这个方案在当前还没能完全实现。一旦协议达到一个更成熟的阶段并且保证没有任何的头部变化（或者只是少量），
    那么我们将会提供一个精简后的二进制数据格式。

当头部的类型为json = 0时，某些优化措施将会用于精简头部的长度。

    头部的字段key为两字节（bytes）长。
    
    如果一个头部的字段值为其默认值，那么它的字段将不包括那些编码为十六进制的字符串值。
    
    标志位在字段‘fg’中编码为双字符的十六进制字符串。

#### 包头的数据字段

	ri: raet id Default 'RAET'
	vn: Version (Version) Default 0
	pk: Packet Kind (PcktKind)
	pl: Packet Length (PcktLen)
	hk: Header kind   (HeadKind) Default 0
	hl: Header length (HeadLen) Default 0
	
    se: Source Estate ID (SEID)
	de: Destination Estate ID (DEID)
	cf: Correspondent Flag (CrdtFlag) Default 0
	bf: BroadCast Flag (BcstFlag)  Default 0

	si: Session ID (SID) Default 0
	ti: Transaction ID (TID) Default 0
	tk: Transaction Kind (TrnsKind)

	dt: Datetime Stamp  (Datetime) Default 0
	oi: Order index (OrdrIndx)   Default 0

	wf: Waiting Ack Flag    (WaitFlag) Default 0
    Next segment or ordered packet is waiting for ack to this packet
	ml: Message Length (MsgLen)  Default 0
    Length of message only (unsegmented)
	sn: Segment Number (SgmtNum) Default 0
	sc: Segment Count  (SgmtCnt) Default 1
	sf: Segment Flag  (SgmtFlag) Default 0
    This packet is part of a segmented message
	af: All Flag (AllFlag) Default 0
    Resend all segments not just one

	bk: Body kind   (BodyKind) Default 0
	ck: Coat kind   (CoatKind) Default 0
	fk: Footer kind   (FootKind) Default 0
	fl: Footer length (FootLen) Default 0

	fg: flags  packed (Flags) Default '00' hs
    2 char Hex string with bits (0, 0, af, sf, 0, wf, bf, cf)
    Zeros are TBD flags
     
#### 正文的数据格式

正文的.data部分是一个使用JSON或MSGPACK序列化好的映射。

#### 数据包的组成成分

每个数据包有4个组成部分，而其中一些可能为空，它们分别是：

    Head
    Body
    Coat
    Tail

头部是必要的部分，其提供数据包处理所需的各种头部字段。

尾部提供用于识别数据包源的认证签名，从而表明它的内容没有被篡改过。

正文部分是整个数据包的内容上下文，一些诸如ACKs和NACKs的数据包一般不需要正文部分。一般来说，正文是已经序列化并排序好的Python字典数据，正文中排序好的数据字段能够让解析和调试拥有一个一致的视图。

表层部分是正文的加密版本，其加密类型是基于CurveCP实现的。如果一个数据包存在Coat部分，那么其正文将会被封装在数据包的表层。

#### 包头的具体细节

##### Json编码格式

包头是一个ASCII安全的JSON编码格式的有组织的Python字典。包头的末端是一个用以两对回车换行符开头的空行。

/r/n/r/n 10 13 10 13 ADAD 1010 1101 1010 1101

回车换行符及换行字符一般不能出现在JSON编码中，除非它们用反斜杠分隔开，因此在一个合法的JSON格式数据中不能出现4字节的组合，因为他们没有多字节的unicode字符使其成为一个唯一的数据包头终止符。

这就意味着包头必须是ASCII安全编码以至于其不允许出现多字节的utf-8格式的字符串。

##### RAET本地编码格式

RAET本地编码格式的数据包头由换行分隔的数据行组成。而包头的每一行都包含一个两字符的字段标识符，紧接着是一个空格跟这个字段的ASCII十六进制编码的二进制数据，后面再跟一个换行符。包头的末尾用一个空行表示，换句话说，就是一对换行符。

##### 二进制编码格式

其包头由一个预定义好的固定长度的字段集合组成。

##### 会话

会话控制对于一个系统的安全性至关重要。Raet希望一个会话打开后，诸多的消息事务均在会话中完成。

会话ID SID称为si

##### 快速开启一个会话

### 分层:

OSI 层次模型

7: 应用层: 
格式: 数据 (应用接口数据缓冲栈等)

6: 表示层: 
格式: 数据 (加密-解密成独立的主机数据格式) 

5: 会话层: 
格式: 数据 (主机间通信，身份验证，组)

4: 传输层: 
格式: 数据段 (消息的可靠传输, 事务, 分段, 错误检测)

3: 网络层: 
格式: 数据包/数据流 (定位和路由)

2: 链路层: 
格式: 数据帧 (保证每一帧的通信连接的可靠性，介质访问控制层）

1: 物理层: 数据位 (通信连接的数据传输并不可靠)
   
   * 数据链路层对于Raet来说是透明的;

   * 网络层包含主机的IP地址和UDP端口信息;

   * Raet在传输层通过事务、数据包认证和尾部签名的相互搭配提供了可靠的数据传输;

   * 会话层便是会话ID为完成签名而进行的密钥交互认证，分组便形成了Road;

   * 表示层完成对数据包正文的加密-解密及序列化-反序列化工作。

   * 应用层便是正文的数据字典。

数据包的签名和认证技术实现上来说可以在传输层或者会话层完成。

### UXD消息

RAET UXD 消息（每个分段）被限制在与RAET UDP消息同等尺寸大小（大概16Mb）。

UXD消息可以有如下的数据包头格式并紧接着一个序列化好的消息正文字典，然而当前仅有JSON数据的格式得以实现。

1) JSON 头部: “RAET\njson\n\n” 紧接着一个统一JSON格式的消息正文的数据字典;

2) msgpack 头部: “RAET\npack\n\n”紧接着一个统一的MSGPACK格式的消息正文的数据字典。
