---
title: "[ 翻译 ] Kubernetes 计划在即将发布的 1.24 版本里弃用 dockershim"
date: 2022-01-22T10:00:00+08:00
tags: ["translate", "kubernetes", "dockershim"]
categories: ["devops"]
comments: false
showMeta: false
showAction: false
---

【编者的话】Kubernetes 计划在即将发布的 1.24 版本里弃用并移除 dockershim

<!--more-->

Kubernetes 计划在[即将发布的 1.24 版本](https://kubernetes.io/blog/2022/01/07/kubernetes-is-moving-on-from-dockershim/)里弃用并移除 dockershim 。使用 Docker 引擎作为其 Kubernetes 集群的容器运行时的工作流或系统需要在升级到 1.24 版本之前进行迁移。1.23 版本将会保留 dockershim，对该版本的支持则会再延长一年。

Docker 是 Kubernetes 使用的第一个容器运行时。最开始对于 Docker 的支持部分被硬编码在 Kubernetes 的代码里，但是随着项目的发展，Kubernetes 开始添加更多的运行时支持。Kubernetes 社区决定转向更加结构化和标准化的接口，而不是直接把第三方的解决方案集成到核心代码里。这就有了[容器运行时接口（CRI）](https://github.com/kubernetes/cri-api)，[容器网络接口（CNI）](https://github.com/containernetworking/cni)以及[容器存储接口（CSI）](https://github.com/container-storage-interface/spec)这些标准。

正如 Mirantis 的 CTO [Adam Parco](https://www.linkedin.com/in/adamparco/) [所说](https://www.mirantis.com/blog/mirantis-to-take-over-support-of-kubernetes-dockershim-2/)：

> 对于大多数人来说，弃用 dockershim 不是什么问题，因为他们甚至可能都感觉不到，他们实际上并没有使用 Docker 本身，他们用的是符合 CRI 标准的 containerd。对于那些人来说，这跟之前没什么两样。

由于 Docker 不符合 CRI 标准，dockershim 更多充当的是 kubelet 和 Docker 之间的一个翻译层。然后 Docker 再通过接口形式调用 containerd 来执行容器。Containerd 之前是作为一个自包含的容器运行时从 Docker 项目里提取出来，随后它便成为第一个符合 CRI 标准的运行时。可以看到，弃用 dockershim 以后 kubelet 便可以和 containerd 这样的容器运行时直接通信。

![picture1](https://imgopt.infoq.com/fit-in/1200x2400/filters:quality(80)/filters:no_upscale()/news/2022/01/kubernetes-dockershim-removal/en/resources/1cri-containerd-1642536386693.png)

*分别采用 containerd 和 dockershim 的 Kubernetes 工作流对比 (来源: [Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/check-if-dockershim-deprecation-affects-you/))*

正如 Kubernetes 博客上[最近的一篇文章](https://kubernetes.io/blog/2020/12/02/dockershim-faq/)所提到的，从 dockershim 迁移出去是为了让 Kubernetes 的代码库能够更好地和新的接口模型保持一致。一些新开发的功能，比如 cgroups v2 还有用户命名空间，它们和 dockershim 在很大程度上是不兼容的。正如最近这篇[博客文章](https://kubernetes.io/blog/2022/01/07/kubernetes-is-moving-on-from-dockershim/)的作者所说："对 Docker 和 dockershim 的依赖已经渗透到 CNCF 生态系统的各种工具和项目里，这会导致我们的代码变得更加脆弱。"

如果你当前使用的是 Docker 来构建应用容器的话，那这些容器仍然可以在其他的容器运行时上运行。但是一旦使用替代的容器运行时，Docker 的一些命令可能就不起作用了或者会产生不同的结果。举个例子，`docker ps` 或者 `docker inspect` 无法再获取容器的信息了，`docker exec` 也不工作了。

在[梳理对 dockershim 的依赖](https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/check-if-dockershim-deprecation-affects-you/)的时候还有一些额外的注意事项，这包括需要确保没有用到特权 Pod 来执行一些 Docker 命令，比如 `docker ps`，重启 Docker service，或者是修改 Docker 的一些特定文件。我们还需要留意一下 Docker 配置文件里有没有私有镜像仓库或者镜像同步源的设置。如果有的话，其他运行时也需要重新配置一下这些设置。

我们还应该检查跑在 Kubernetes 基础设施之外的一些脚本，识别出用到 Docker 命令的部分。这也许包括 SSH 到机器上排障，节点的启动脚本，又或者是直接装在节点上的一些[监控和安全客户端](https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/migrating-telemetry-and-security-agents/)。

Mirantis 和 Docker 已经达成一致，他们将会共同维护 dockershim 的代码，后续它将作为一个相对于 Kubernetes 而言更加独立、[开源](https://github.com/Mirantis/cri-dockerd)并且符合 CRI 接口的项目存在。这意味着 Mirantis 容器运行时（MCR）将会是符合 CRI 标准的运行时。如果遇到不希望或者不能接受 dockershim 下线的情况，他们的 [cri-dockerd](https://github.com/Mirantis/cri-dockerd) 将可以用作 dockershim 的一个外部替代品。

Kubernetes 1.24 版本的发布团队和 CNCF 已经承诺了将会及时提供此次发布相关的文档，目前计划是在 4 月份。这里面包含了博客文章，更新代码示例，教程，以及一份迁移指南。

该团队认为，继续进行迁移已经不存在任何主要障碍。他们确实有注意到，最近的 11 月 11 日 [SIG Node 兴趣小组](https://docs.google.com/document/d/1Ne57gvidMEWXR70OxxnRkYquAoMpt56o75oZtg-OeBg/edit#bookmark=id.r77y11bgzid)以及 12 月 6 日 [Kubernetes 指导委员会](https://docs.google.com/document/d/1qazwMIHGeF3iUh5xMJIJ6PDr-S3bNkT8tNLRkSiOkOU/edit#bookmark=id.m0ir406av7jx)会议上有关推迟弃用 dockershim 的讨论。然而，此时他们已经同意继续推进在即将发布的 1.24 版本里移除 dockershim 的工作。

**原文链接：[Kubernetes Proceeding with Deprecation of Dockershim in Upcoming 1.24 Release](https://www.infoq.com/news/2022/01/kubernetes-dockershim-removal/)（翻译：吴佳兴)**
