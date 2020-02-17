---
title: "[ 翻译 ] 开源世界的\"服务发现\""
date: 2014-08-30T20:00:00+08:00
tags: ["devops", "cloud-native", "discovery", "translate"]
categories: ["cloud-native"]
comments: false
showMeta: false
showAction: false
---

这里是对服务发现的一篇翻译，原文见：[Open-Source Service Discovery](http://jasonwilder.com/blog/2014/02/04/service-discovery-in-the-cloud/)

<!--more-->

服务发现是众多分布式系统和面向服务的架构的一个关键组件。它所解决的问题乍看上去好像挺简单：客户端怎么在现存的众多主机群里定位到一个服务的IP和对应端口?

一般来说，如果你使用一些静态的配置来完成这项任务，那可能之后的维护会有点费劲。在你开始部署更多的服务时事情也许会变得更加复杂。一个运转中的系统，由于经常进行自动或主动的扩展的原因，它所提供的各种服务的“地理位置”可能会发生频繁的变换，包括新部署的一些服务，以及被替换掉的故障主机。

动态的服务注册和发现在这样的情境下显得尤为突出，旨在通过其避免服务的中断。

针对这个问题，业界已经提出了多套不同的解决方案，而且还在持续不断的发展和深入。我们来看看一些开源的或者公开讨论过的针对该问题的解决方案，学习下它们是怎样工作的。特别地，我们将会来看一看每个解决方案在存储、运行时依赖、客户端集成方面的强\弱一致性，以及针对这些特性是怎样权衡的。

我们首先会以一些诸如Zookeeper、Doozer和Etcd这样的强一致性的项目开始讲述，它们一般用于协调服务，但是也可以用于注册服务。

我们之后将会了解一些有趣的解决方案，它们专门设计用于服务注册和发现。我们将涉及到Airbnb的SmartStack, Netflix的Eureka, Bitly的NSQ, Serf, Spotify和DNS，最后便是SkyDNS。

#### 问题所在?

定位服务存在两方面的问题，那便是服务注册和服务发现。

* **服务注册** - 在中央注册节点登记一个服务所在位置信息的过程。通常会记录下它的主机名和端口，有时候可能还需要认证证书、协议、版本号、以及\或者更多详细的环境信息。

* **服务发现** - 客户端应用程序从中央注册节点查询并获取到所需服务的地址信息的过程。

任何一个服务注册与发现的解决方案同时都要考虑其他的开发以及运维方面的需求：

* **监控** - 当一个已注册的服务发生故障时会怎样? 有时候它不会立刻被反注册，也许等到一个timeout的时间周期（或其他处理逻辑）之后才会如此。服务通常需要实现一个心跳机制来证明存活状态而客户端一般需要能够处理故障服务的可靠性问题。

* **负载均衡** - 如果多个服务被注册在案，客户端们该如何去均衡访问这些服务的负载呢? 如果存在一个master的角色，它能够被一个客户端正确的命中吗？

* **集成风格** - 中央注册节点只能提供一些有限的程序语言的监听吗，例如，仅仅支持Java? 服务注册和发现的整合需要额外在你的应用程序里嵌入一些注册和发现的代码或者使用*"伙伴进程"*也是一个选择?

* **运行时依赖** - 它是否需要JVM、Ruby或者一些不兼容于你的环境的依赖项?

* **可用性的考虑** - 你能否在丢失一个节点的情况下正常工作? 它能否平滑升级而无需无谓的宕机时间?注册节点可能会成为你系统架构中的一个中央组成部分，而且可能会引发单点故障。

#### 通用注册节点

这里前三个注册中心使用的是强一致性的协议而且实际上是通用目的的、一致性的数据存储。虽然我们将他们视为服务注册中心，但是他们也常用于服务的协调，指明“领导选举”或者分布式客户端的集中式锁定。

##### Zookeeper

Zookeeper 是一个集中式服务，用来维护配置信息、域名解析以及分布式同步，并且提供群组服务。它使用Java编写，是强一致性的，并且使用Zab协议来协同集群(ensemble)之间的更改。

Zookeeper通常在一个ensemble里运行三个、五个或者七个的成员，客户端为了访问ensemb，需要使用特定的语言去监听。访问方式通常嵌入到客户端程序和服务代码里。

服务注册是通过一个命名空间下的 [ephemeral节点](http://zookeeper.apache.org/doc/r3.2.1/zookeeperProgrammers.html#Ephemeral+Nodes) 实现的。 Ephemeral节点仅在客户端连接时有效，所以通常是在一个后端服务注册在启动完成后注册自身地址信息的时候存在。如果它故障或者断开连接，那么这个节点便从服务树中消失。

服务发现是通过列出和查看指定服务的命名空间实现的。客户端接收到所有的当前的已注册服务，并会在一个服务变为不可用或者一个新的服务注册进来的时候收到提醒消息。客户端也需要处理一些负载均衡或者自身的故障恢复。

Zookeeper的API很难被正确合适的使用，而程序语言的监听可能存在一些细微的差别，这可能导致一些问题的产生。如果你在使用基于JVM的程序语言，这个插件[Curator Service Discovery Extension](http://curator.apache.org/curator-x-discovery/index.html)可能会有用武之地。

由于Zookeeper自身定义为一个强耦合的系统，当一个[Partition](http://wiki.apache.org/hadoop/ZooKeeper/FailureScenarios)发生时，在Partition期间系统里的一些服务将不能再被注册或者就算他们能正常工作也无法完成“定位现有注册服务”的动作。特别地，在任何不合数的情况下，读写都会报错。

##### Doozer

[Doozer](https://github.com/ha/doozerd)同样是一个一致性的、分布式的数据存储。它用Go语言编写，其为强一致性的并且使用[Paxos](http://research.microsoft.com/en-us/um/people/lamport/pubs/lamport-paxos.pdf)来维护一致性。这个项目历时数年但是有所停滞，目前仅有接近160次的fork。不幸的是，这让我们很难知道项目现在的发展状况以及是否可以用于生产环境。

Doozer一般运行在3、5或者7个节点的集群。客户端使用程序语言指定监听来访问集群，并且，同Zookeeper类似，代码被整合嵌入到客户端和服务的代码里。

服务注册则并不像Zookeeper那样直截了当，因为Doozer并没有ephemeral nodes的概念。一个服务能通过一个路径去注册本身，但是如果它变为不可用的状态，其不会被自动的删除。

这里有很多途径可以定位到这样的问题。其中一种可以是添加时间戳和心跳机制到注册进程，并且在服务发现的过程或是其他清理过程期间处理过期的条目。

服务发现则和Zookeeper类似，你可以列出所有某个路径下的所有条目，并且之后可以接收到该路径下的任意变更。如果你在注册时使用时间戳和心跳的方式，在服务发现过程中你将不用再关心或者删除任何过期的条目。

像Zookeeper一样，Doozer也是一个CP系统（强一致性系统）而且在Partition发生时同样会受到影响。

##### Etcd

[Etcd](https://github.com/coreos/etcd) 是一个针对共享配置和服务发现的高可用的、键-值对存储系统。Etcd受到Zookeeper和Doozer的启发。它用Go语言编写，使用[Raft](https://ramcloud.stanford.edu/wiki/download/attachments/11370504/raft.pdf)保证一致性并且拥有一个基于Http + JSON的API。

Etcd，类似于Doozer和Zookeeper，通常运行在3、5或者7个节点的集群环境。客户端使用特定的语言去监听或者也可以通过一个HTTP客户端实现。

服务注册依赖于通过探测特定key的服务一个TTL时间内的心跳响应来验证相应的key是否依旧可用。如果一个服务没有及时的更新对应key的TTL，那么Etcd便会将其判定为过期。如果一个服务变为不可用，客户端将需要拥有处理连接故障的能力并且可以尝试其他的服务实例。

服务发现包括列出目录下所有的key，以及在目录内容改变时能接收到消息。由于其API是基于HTTP协议，客户端程序只需要保持一个和Etcd集群的长轮询连接即可。

自Etcd使用Raft以来，它便一直是一个强一致性系统的定位。Raft需要一个被选举出来的领导者而所有的客户端都要被领导者调度。然而，通过这个参数[undocumented consistent parameter](https://github.com/coreos/etcd/blob/master/server/v2/get_handler.go#L25)，Etcd似乎同样支持non-leaders模式的read方式，其将可以提高读方面的可用性。写模式在一个partition出现并且可能故障的情况下，则仍然需要一个领导者角色来做相应的抉择。

#### 单一目标的注册中心

下面几个注册服务设计上特别适用于服务注册和发现。大多数来自于实际的生产用例，而其他一些则是针对该领域问题的比较有意思以及差异化的解决方案。相比之下，Zookeeper、Doozer和Etcd也可以用于分布式的协调服务，这些解决方案却不具备这样的能力。

##### Airbnb的Smartstack

Airbnb的[Smartstack](http://nerds.airbnb.com/smartstack-service-discovery-cloud/)是两个独立工具的结合，[Nerve](https://github.com/airbnb/nerve)和[Synapse](https://github.com/airbnb/synapse),其利用[HAProxy](http://haproxy.1wt.eu)和[Zookeeper](http://zookeeper.apache.org/)来处理服务注册和服务发现。Nerve和Synapse都是用Ruby编写。

Nerve是以一个 *伙伴进程* 的方式存在，其作为一个独立于应用程序的服务运行。Nerve负责完成在Zookeeper的服务注册工作。应用程序暴露一个/heath节点，作为一个HTTP服务，这样Nerve便可以持续的完成监控。提供的服务是可用的，它将会被预先在Zookeeper完成注册。

*伙伴* 模式使得服务无需再与Zookeeper进行交互。它简单的只需要一个监控的端点来完成注册。这使得它更容易通过不同的语言去支持服务，而这可能造成Zookeeper的监听不大健壮。[好莱坞原则](http://en.wikipedia.org/wiki/Hollywood_principle)也提供了诸多的优势。

Synapse同样是一个独立于服务运行的伙伴进程。它负责服务发现。其依靠查询Zookeeper获取现有注册服务并且重新配置本地运行的HAProxy实例来完成这样一个任务。任意运行在主机上的客户端倘若需要访问其他的服务，那么便会经由本地的HAProxy实例路由该请求到一个可用的服务。

Synapse的设计简化了服务的实现，这样一来，他们便不再需要实现任何客户端方面的负载均衡和故障恢复，而且他们也不再需要依赖Zookeeper或者其提供的语言监听。

由于Smartstack依赖于Zookeeper，一些注册和发现任务将会在Partition的时候失败。他们指出Zookeeper是他们设定上的“阿喀琉斯之踵”（薄弱环节）。所提供的服务在Partition之前能够至少一次的发现其他可替代的服务，在Partition之后它应该还保留一个服务的快照，而在Partition期间它也许能够持续的运转。这方面可以提高整个系统的可用性和可靠性。

更新：如果你对Smartstack风格的Docker容器解决方案有兴趣，可以看看这篇文章[Docker service discovery](http://jasonwilder.com/blog/2014/07/15/docker-service-discovery/)。

##### Netflix的Eureka

[Eureka](https://github.com/Netflix/eureka)是Netflix的中间层，承载负载均衡和服务发现任务。其拥有一个服务组件以及其使用的智能客户端应用服务。服务器和客户端都使用Java编写，这意味着其最理想的用例将会是在同样使用JAVA或其他JVM兼容的语言实现的服务上。

Eureka服务器是服务的注册中心。他们建议一个Eureka服务器理应运行在每个AWS的可用区域从而形成一个集群。这些服务器互相通过一个异步模型复制其状态到其余的节点，这意味着在指定时间里每个实例可能给出各自略有差异的针对所有注册服务的描述画面。

服务的注册通过客户端组件处理。服务内嵌客户端到他们的应用程序代码里。在运行时，客户端注册其服务并且定期的发送心跳包来刷新它的“状态租约”。

服务的发现则是通过智能客户端来处理。它从服务器以及他们的本地cache来检索当前的注册项。客户端定期的刷新其状态，同时也处理相应的负载均衡和故障恢复任务。

Eureka在故障期间的处理设计的非常具有灵活性。它支持通过强耦合性保持可用性，并且能够在一定数量的不同故障模式下持续的运转。如果这里出现一个集群的Partition，Eureka将会过渡到一个自我保护状态。在Partition期间其允许服务依旧可以被正常的发现和注册，而当其恢复时，集群所属的成员将会重新合并他们的状态。

##### Bitly 的 NSQ lookupd

[NSQ](https://github.com/bitly/nsq)是一个实时的、分布式的消息平台。它使用Go语言编写，并且提供一个基于HTTP的API。虽然它不是一个通用目的的服务注册和发现工具，但是为了客户端能够具备在运行时发现[nsqd](http://bitly.github.io/nsq/components/nsqd.html)实例的能力，他们在其[nsqlookupd](http://bitly.github.io/nsq/components/nsqlookupd.html)客户端中编写了一个服务发现的novel模型。

在一个NSQ的部署过程中，nsqd本质上便是一个服务。这些便是消息的存储。nsqlookupd是一个服务的注册中心。客户端程序直接连接到nsqd实例，一旦他们在运行时出现更改，客户端可以通过检索nsqlookupd实例来发现可用的实例。

对于服务注册，每个nsqd实例定期地发送一个指明其状态的心跳包到每个nsqlookupd实例。他们的状态包含地址信息以及其他队列信息或者他们所需要关注的话题。

对于服务发现，客户端查询每个nsqlookupd实例并且将所得的结果合并做出决策。

这个模型有意思的一点在于其nsqlookupd实例并不知道其他的同等存在。客户端的责任在于合并从每个单个的nsqlookupd实例返回的状态数据并从而确定整体的状态。由于每个nsqd实例通过心跳包来反馈它的状态，所以每个nsqlookupd最终能够提供一套相同的每个nsqd实例能够连接的所有可用的nsqlookupd实例信息。

前面所讨论的所有注册中心组件整个组成一个集群并且使用一个强一致性或弱一致性的协议来维护它们的状态。NSQ的设计本质上是弱一致性的，但是也能容忍Partition。

##### Serf

[Serf](http://www.serfdom.io/)是一个非集中式的服务发现与编制的解决方案。它同样是用Go语言编写，并且是唯一一个使用基于gossip的协议，[SWIM](http://www.cs.cornell.edu/~asdas/research/dsn02-swim.pdf)，来完成成员管理、故障检测和自主的event分发。SWIM设计之初是为了解决传统心跳协议的不可扩展性问题。

Serf包括一个简单的二进制文件，其需要被安装到每个主机上，而它能够运行作为一个agent加入或创建一个集群，或者作为一个client这样它便可以主动发现集群内的成员。

针对服务注册，serf的agent将会被运行并加入到一个现有的集群里。agent在启动时附上一些自定义的tag从而可以定位主机角色、环境、IP、端口等等信息。一旦加入到集群里，其他的成员将能够“看”到这台主机和它的元数据。

针对服务发现，serf通过运行`members`命令获取集群里的现有成员。借助members的输出，过滤他们运行中的agent所表明的tag，你能够发现到提供特定服务的所有主机，。

Serf是一个相对新兴的项目而且变革很快。在这篇文章里这是唯一一个没有使用中央注册架构风格的项目，而这令它变得十分独特。自其使用异步的，基于gossip的协议起，它就注定是弱一致性的但是更具备容错性和可用性。

##### Spotify 和 DNS

Spotify在他们的一篇文章[In praise of "boring" technology](http://labs.spotify.com/tag/dns/)里描述了他们是如何使用DNS来完成服务发现的。不使用一个新的技术，取而代之的是选择在DNS的上层建立一个更为成熟的技术架构。Spotify将DNS解读为“分布式的，可复制的适用于重度读负载的数据库”。

Spotify使用一个相对未知的[SRV记录](en.wikipedia.org/wiki/SRV_record)来完成服务发现的意图。SRV记录能够看做是更抽象化的MX记录。他们允许你去定义一个服务的名称，协议，TTL，优先级，权重，端口和目标主机。基本上，便是一个客户端需要的全部信息用以定位所有可用的服务和完成负载均衡。

由于他们使用源代码来管理所有的区域文件，所以其服务的注册是相对复杂和静态的配置过程。服务发现则是使用一些差异化的DNS客户端库或是自定义的第三方工具来完成。他们也在自身服务本地保留DNS的cache，借此来尽可能的减少根DNS服务器的负载。

在他们的文章最后提及这个模型虽然工作的很好，但是他们仍然在尝试扩展，并着手研究Zookeeper从而支持静态\动态双方面的服务注册。

##### SkyDNS

[SkyDNS](https://github.com/skynetservices/skydns)是一个相对新兴的使用Go语言编写实现的项目，使用RAFT来保证一致性而且同样提供一个基于HTTP和DNS的客户端API。它和Etcd以及Spotify的DNS模型有一些相似之处，而且实际上和Etcd、[go-raft](https://github.com/goraft/raft)一样使用RAFT实现。

SkyDNS 服务器同样是一个集群的概念，使用RAFT协议，并且需要选举出一个leader角色。SkyDNS服务器为服务注册和发现暴露不同的访问端点。

针对服务注册，服务均是使用一个基于HTTP的API来创建一个带TTL值的条目。服务必须持续不断的发出心跳包来验证其状态。SkyDNS也使用SRV记录但是将它们扩展成支持服务版本，环境和区域等特性。

针对服务发现，客户端使用DNS并且检索SRV记录来定位其需要连接的服务。客户端需要实现任意的负载均衡或者故障恢复，并且将可能在本地cache以及定期的刷新服务位置数据。

与Spotify使用DNS的方式有所不同，SkyDNS支持动态的服务注册并且能够独立的完成该任务而无需依赖其他额外的第三方诸如Zookeeper或者Etcd的服务。

如果你正在使用[Docker](docker.io)，[skydock](https://github.com/crosbymichael/skydock)也许值得你checkout从而使得SkyDNS自动的集成到你的容器中。

总的来说，这是一个旧技术（DNS）和新技术（Go，RAFT）的有趣结合，并且观察之后该项目的发展去向将会是一件蛮有意思的事情。

#### 总结

在上文，我们了解了一系列的有通用目标的，强一致性的注册中心（Zookeeper，Doozer，Etcd）和一些自主构建的，最终一致性的项目（Smartstack，Eureka，NSQ，Serf，Spotify的DNS，SkyDNS）。

许多可能使用嵌入式的客户端库件（Eureka，NSQ等等）而其他一些可能使用独立的伙伴进程（Smartstack，Serf）。

有趣的是，一些专用的解决方案，他们都采用了一种可用性高过一致性的设计思路。

![图1-1 各开源的服务注册-发现组件的比较](/images/2014/Aug/tu-1.png)

（1）如果使用`一致性`参数，便可能出现非一致性的读；
（2）如果在SkyDNS之前使用一个缓存的DNS客户端，读可能变得不一致。
