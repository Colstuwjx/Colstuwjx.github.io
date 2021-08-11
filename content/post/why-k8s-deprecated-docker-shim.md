---
title: "【源码解读】从代码实现层面思考 Kubernetes 为什么会弃用对 Docker 的支持？"
date: 2021-08-11T10:00:00+08:00
tags: ["cloudnative", "kubernetes", "docker"]
categories: ["cloudnative"]
comments: false
showMeta: false
showAction: false
---

2020年底，在 Kubernetes v1.20 正式发布的同时，k8s 官方还搞了一个大动作：他们宣布将会逐步弃用对 Docker 容器运行时的支持。为了不让用户惊慌失措，官方还贴心地写了一篇[博客文章](https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/)，对此事进行了一番详细说明。

> K8s 为什么会弃用对 Docker 的支持呢？

除了官方的这篇文章以外，很多科技媒体也做了相应的解读，比如 infoq 的这篇[文章](https://www.infoq.cn/article/47hcixefry1cetbzugwd)。这里，笔者并不想做过多重复的泛泛而谈，更多地是希望能够站在社区用户的角度来看待这件事情，基于代码实现层面捋一捋这件事情的来龙去脉，希望能给读者朋友们带来一些启发。

<!--more-->

## 前世

在官方发布的博客文章里链接了一份[弃用 Dockershim 的常见问题解答](https://kubernetes.io/blog/2020/12/02/dockershim-faq/)。在这份 FAQ 里，官方也提到了弃用 Dockershim 的根本原因：

`Docker itself doesn't currently implement CRI, thus the problem. Dockershim was always intended to be a temporary solution (hence the name: shim).`

翻译一下就是: Dockershim 是当初 k8s 引入 CRI 容器运行时标准接口的时候为了兼容 Docker，k8s 官方自行维护的一套临时解决方案，他们现在不想再维护了。

### dockershim 的起点

为了印证这一点，笔者找到了当初开发人员提的第一个[ PR ](https://github.com/kubernetes/kubernetes/pull/29553)，我们还可以根据 PR 里给出的信息，顺藤摸瓜，找到相关的[ umbrella issue ](https://github.com/kubernetes/kubernetes/issues/28789)。为了让 Docker 支持 CRI 标准，核心开发 [yujuhong](https://github.com/yujuhong) 也贡献了大量的集成代码（也即是 dockershim 的实现），具体可以查阅[这个 issue ](https://github.com/kubernetes/kubernetes/issues/31459)。

*注：dockershim 的代码位于 k8s 仓库的[这里](https://github.com/kubernetes/kubernetes/tree/v1.22.0/pkg/kubelet/dockershim)，我们可以很方便地通过[追溯 commit history ](https://github.com/kubernetes/kubernetes/commits/master?after=5a732dcfe1d4ec0e8ee2871b106605b7f8a69b98+104&branch=master&path%5B%5D=pkg&path%5B%5D=kubelet&path%5B%5D=dockershim&path%5B%5D=docker_service.go)来找到首次提交，最终便找到了这个[ PR ](https://github.com/kubernetes/kubernetes/pull/29553)。*

### 前 CRI 时代

在翻阅这块代码的时候，笔者还有一个疑问: **在 CRI 标准提出之前，k8s 是怎么和容器运行时交互的呢？**

带着这个问题，笔者通过搜索找到了宣布引入 CRI 标准的[官方文章](https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/)。

文章里介绍到，自 k8s 1.5 起，CRI 功能作为一个 alpha 特性被引入到 k8s，而早在 1.3 版本开始，k8s 就集成了 rkt 容器运行时的支持，作为替代 docker 的可选方案。

然而，这些代码都是托管在 k8s kubelet 的核心代码里，后续维护和增加更多容器运行时支持都会变得越来越困难，这也是 k8s 为什么会提出一个抽象的 CRI 标准来和实际的具体实现做解耦的原因。

那么，我们不妨再来看看当时版本的 k8s 具体是怎么和 docker 及 rkt 引擎做交互的吧。

定位代码的方式也很简单，我们直接选择 1.5 之前的版本，比如 v1.4.0 的 tag 版本。然后，既然官方说代码嵌在了 kubelet 代码里，我们可以直接切到[ kubelet 代码的目录](https://github.com/kubernetes/kubernetes/blob/v1.4.0/pkg/kubelet)，不难找到下面这两个子目录：

* [rkt](https://github.com/kubernetes/kubernetes/tree/v1.4.0/pkg/kubelet/rkt)

* [dockertools](https://github.com/kubernetes/kubernetes/tree/v1.4.0/pkg/kubelet/dockertools)

*注：这里还有一个小彩蛋，我们在该目录下还找到了一个[ rktshim ](https://github.com/kubernetes/kubernetes/tree/v1.4.0/pkg/kubelet/rktshim)的目录，说明当初各个容器运行时尚未普及对 CRI 的支持时，在 k8s 代码里嵌 xxxshim 服务用作临时支持是一个常规操作。*

> 那么，它们到底是咋交互的呢？

我们不难在 dockertools 目录下的 kube_docker_client.go 里面找到一个[ kubeDockerClient ](https://github.com/kubernetes/kubernetes/blob/v1.4.0/pkg/kubelet/dockertools/kube_docker_client.go#L38)的实现：

```
// kubeDockerClient is a wrapped layer of docker client for kubelet internal use. This layer is added to:
//	1) Redirect stream for exec and attach operations.
//	2) Wrap the context in this layer to make the DockerInterface cleaner.
//	3) Stabilize the DockerInterface. The engine-api is still under active development, the interface
//	is not stabilized yet. However, the DockerInterface is used in many files in Kubernetes, we may
//	not want to change the interface frequently. With this layer, we can port the engine api to the
//	DockerInterface to avoid changing DockerInterface as much as possible.
//	(See
//	  * https://github.com/docker/engine-api/issues/89
//	  * https://github.com/docker/engine-api/issues/137
//	  * https://github.com/docker/engine-api/pull/140)
// TODO(random-liu): Swith to new docker interface by refactoring the functions in the old DockerInterface
// one by one.
type kubeDockerClient struct {
	// timeout is the timeout of short running docker operations.
	timeout time.Duration
	client  *dockerapi.Client
}
```

上面的注释也写的挺详细，大致意思就是它是一个和 Docker 交互的 client，并封装了一些 k8s 操作 Docker 需要的一些接口方法，这套接口方法具体定义在同一目录的 docker.go 里：

```
// DockerInterface is an abstract interface for testability.  It abstracts the interface of docker client.
type DockerInterface interface {
	ListContainers(options dockertypes.ContainerListOptions) ([]dockertypes.Container, error)
	InspectContainer(id string) (*dockertypes.ContainerJSON, error)
	CreateContainer(dockertypes.ContainerCreateConfig) (*dockertypes.ContainerCreateResponse, error)
	StartContainer(id string) error
	StopContainer(id string, timeout int) error
	RemoveContainer(id string, opts dockertypes.ContainerRemoveOptions) error
	InspectImage(image string) (*dockertypes.ImageInspect, error)
	ListImages(opts dockertypes.ImageListOptions) ([]dockertypes.Image, error)
	PullImage(image string, auth dockertypes.AuthConfig, opts dockertypes.ImagePullOptions) error
	RemoveImage(image string, opts dockertypes.ImageRemoveOptions) ([]dockertypes.ImageDelete, error)
	ImageHistory(id string) ([]dockertypes.ImageHistory, error)
	Logs(string, dockertypes.ContainerLogsOptions, StreamOptions) error
	Version() (*dockertypes.Version, error)
	Info() (*dockertypes.Info, error)
	CreateExec(string, dockertypes.ExecConfig) (*dockertypes.ContainerExecCreateResponse, error)
	StartExec(string, dockertypes.ExecStartCheck, StreamOptions) error
	InspectExec(id string) (*dockertypes.ContainerExecInspect, error)
	AttachToContainer(string, dockertypes.ContainerAttachOptions, StreamOptions) error
	ResizeContainerTTY(id string, height, width int) error
	ResizeExecTTY(id string, height, width int) error
}
```

可以看到，k8s 在 Pod 的生命周期里需要用到的一些操作函数都已经包含在内。

到这里，我们可以大致概括一下 k8s 在引入 CRI 阶段的一个迭代过程：

1、在 CRI 标准落地之前，k8s 等于是为每一个容器运行时都实现了一个具体的对接，rkt 和 dockertools 目录下即对应的代码实现；

2、官方于 1.5 版本开始正式引入 CRI 标准，并实现了对应的 shim 代码，如 dockershim 和 rktshim，在各个容器运行时尚未支持 CRI 标准的接口之前，充当一个胶水服务。

## 今生

通过追溯之前的版本历史，我们了解了 k8s 在支持容器运行时这块的"坎坷经历"。

然而，这里还有一个问题始终未能得到解答：**为什么非得要弃用 dockershim ? 继续维护下去的话究竟会有哪些具体的痛点呢？**

毕竟，如果弃用 dockershim 的话，这意味着原本使用 docker 作为容器引擎的用户需要为此计划实施迁移到 containerd 或者其他支持 CRI 的容器运行时，这会是一个不小的时间和人力成本。

想要解答这个问题，恐怕还得先看看 dockershim 目前的使用场景以及 CRI 的发展现状。

### 在 kubelet 启动时运行 dockershim 服务

时至今日，kubelet 要去启动一个 docker 容器的话，究竟是怎么和 dockershim 配合工作的呢？我们不妨再来看看 kubelet 这层的代码实现。

这次我们选用的是刚发布不久的 [v1.22](https://github.com/kubernetes/kubernetes/tree/v1.22.0) 版本的代码。

kubelet 的启动入口位于 [cmd/kubelet/kubelet.go](https://github.com/kubernetes/kubernetes/blob/v1.22.0/cmd/kubelet/kubelet.go#L36)，熟悉 [cobra](https://github.com/spf13/cobra) 的朋友应该知道，它最终是会调用具体 `Command` 的 `Run` 方法。对于 kubelet 来说，调用的即是它实现的 [Run](https://github.com/kubernetes/kubernetes/blob/v1.22.0/cmd/kubelet/app/server.go#L155) 方法。

在经过一系列的处理后，kubelet 会走到核心的用来启动服务的 [run](https://github.com/kubernetes/kubernetes/blob/v1.22.0/cmd/kubelet/app/server.go#L514) 方法。我们直接看向和 Dockershim 相关的部分，划到靠近函数末尾的部分，我们可以看到在真正启动前，kubelet 执行了一个 `[kubelet.PreInitRuntimeService](https://github.com/kubernetes/kubernetes/blob/v1.22.0/cmd/kubelet/app/server.go#L796)` 的操作。

> 这个 PreInitRuntimeService 方法做了什么事情呢？

我们不妨继续深入一下，看看它的[具体内容](https://github.com/kubernetes/kubernetes/blob/v1.22.0/pkg/kubelet/kubelet.go#L297)：

```
// PreInitRuntimeService will init runtime service before RunKubelet.
func PreInitRuntimeService(kubeCfg *kubeletconfiginternal.KubeletConfiguration,
	kubeDeps *Dependencies,
	crOptions *config.ContainerRuntimeOptions,
	containerRuntime string,
	runtimeCgroups string,
	remoteRuntimeEndpoint string,
	remoteImageEndpoint string,
	nonMasqueradeCIDR string) error {
	if remoteRuntimeEndpoint != "" {
		// remoteImageEndpoint is same as remoteRuntimeEndpoint if not explicitly specified
		if remoteImageEndpoint == "" {
			remoteImageEndpoint = remoteRuntimeEndpoint
		}
	}

	switch containerRuntime {
	case kubetypes.DockerContainerRuntime:
		klog.InfoS("Using dockershim is deprecated, please consider using a full-fledged CRI implementation")
		if err := runDockershim(
			kubeCfg,
			kubeDeps,
			crOptions,
			runtimeCgroups,
			remoteRuntimeEndpoint,
			remoteImageEndpoint,
			nonMasqueradeCIDR,
		); err != nil {
			return err
		}
	case kubetypes.RemoteContainerRuntime:
		// No-op.
		break
	default:
		return fmt.Errorf("unsupported CRI runtime: %q", containerRuntime)
	}

    ...
}
```

可以看到，当 `containerRuntime` 参数是 `kubetypes.DockerContainerRuntime` 时，kubelet 需要执行额外的 `runDockershim` 方法去启动一个 dockershim 服务（可以看到，上面有一行警告 dockershim 已弃用的提醒），而如果是 `kubetypes.RemoteContainerRuntime` 类型的话，则什么事情也不用干。

我们可以在 kubelet 目录下找到 `kubelet_dockershim.go`，该文件里即实现了这个 [runDockershim](https://github.com/kubernetes/kubernetes/blob/v1.22.0/pkg/kubelet/kubelet_dockershim.go#L30) 方法，它会去调用 dockershim 的相关服务代码并启动一个 `dockerServer`。

很显然， kubelet 是通过这个 dockershim 服务包装的一层 CRI 接口调用 docker 启动 Pod 容器的。我们不妨看下 kubelet 实际是怎么去起 Pod 的，然后再来看看调用运行时服务这块的实现细节。

### 通过 dockershim 服务 syncPod

TODO.

### containerd beyond 1.0

了解了 kubelet 调用 dockershim 这块的情况以后，我们再来看看它的表兄弟 containerd 的现状。

## 结语

TODO.

## 参考

1、https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#deprecation

2、https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/

3、https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/

4、https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md#design-docs-and-proposals

5、https://opencontainers.org/about/overview/

6、https://dev.to/inductor/wait-docker-is-deprecated-in-kubernetes-now-what-do-i-do-e4m
