---
title: "【源码解读】从代码实现层面思考 Kubernetes 为什么会弃用对 Docker 的支持？"
date: 2021-08-11T10:00:00+08:00
tags: ["cloudnative", "kubernetes", "docker"]
categories: ["cloudnative"]
comments: false
showMeta: false
showAction: false
aliases:
    - /dive-into-sourcecode-why-k8s-deprecated-dockershim/
---

2020年底，在 Kubernetes v1.20 正式发布的同时，k8s 官方还搞了一个大动作：他们宣布将会逐步弃用对 Docker 容器运行时的支持。为了不让用户惊慌失措，官方还贴心地写了一篇[博客文章](https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/)，对此事进行了一番详细说明。


**K8s 为什么会弃用对 Docker 的支持呢？**除了官方的这篇文章以外，很多科技媒体也做了相应的解读，比如 infoq 的这篇[文章](https://www.infoq.cn/article/47hcixefry1cetbzugwd)。但是，为什么一定要弃用 docker 呢？这方面的维护成本究竟有多高？为了得到一个明确的答案，笔者决定展开一次 k8s 源码的探索之旅，一探究竟。

<!--more-->

<!--toc-->

# 前世

在官方发布的博客文章里链接了一份[弃用 Dockershim 的常见问题解答](https://kubernetes.io/blog/2020/12/02/dockershim-faq/)。在这份 FAQ 里，官方也提到了弃用 Dockershim 的根本原因：

`Docker itself doesn't currently implement CRI, thus the problem. Dockershim was always intended to be a temporary solution (hence the name: shim).`

翻译一下就是: Dockershim 是当初 k8s 引入 CRI 容器运行时标准接口的时候为了兼容 Docker，k8s 官方自行维护的一套临时解决方案，他们现在不想再维护了。

## dockershim 的起点

那么，dockershim 是什么时候加进去的呢？当时的背景又是怎样的？

笔者找到了当初开发人员提的第一个[ PR #29553 ](https://github.com/kubernetes/kubernetes/pull/29553)，PR title 里面有这么一句话：

```
yujuhong: ... Add a new docker integration with kubelet using the new runtime API ...
```

根据 PR 里给出的信息，顺藤摸瓜，笔者又找到了相关的[ umbrella issue ](https://github.com/kubernetes/kubernetes/issues/28789)，主要是用来跟踪 CRI 对接的进展。也就是说，为了让 Docker 支持 CRI 标准，核心开发 [yujuhong](https://github.com/yujuhong) 贡献了集成 docker 操作并且支持新版 runtime API 的一个组件（也即是 dockershim ），具体可以查阅[这个 issue ](https://github.com/kubernetes/kubernetes/issues/31459)，这是当时用来跟踪 dockershim 实现 CRI 接口专门开的一个 issue。

*注：dockershim 的代码位于 k8s 仓库的[这里](https://github.com/kubernetes/kubernetes/tree/v1.22.0/pkg/kubelet/dockershim)，我们可以很方便地通过[追溯 commit history ](https://github.com/kubernetes/kubernetes/commits/master?after=5a732dcfe1d4ec0e8ee2871b106605b7f8a69b98+104&branch=master&path%5B%5D=pkg&path%5B%5D=kubelet&path%5B%5D=dockershim&path%5B%5D=docker_service.go)来找到首次提交，最终便找到了这个[ PR ](https://github.com/kubernetes/kubernetes/pull/29553)。*

## 前 CRI 时代

在翻阅这块代码的时候，笔者内心还有一个疑问: **在 CRI 标准提出之前，k8s 是怎么和容器运行时交互的呢？**

带着这个问题，笔者通过搜索找到了宣布引入 CRI 标准的[官方文章](https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/)。

文章里介绍到，自 k8s 1.5 起，CRI 功能作为一个 alpha 特性被引入到 k8s，而早在 1.3 版本开始，k8s 就集成了 rkt 容器运行时的支持，作为替代 docker 的可选方案。

然而，这些代码都是托管在 k8s kubelet 的核心代码里，后续维护和增加更多容器运行时支持都会变得越来越困难。

那么，我们不妨来看看当时版本的 k8s 具体是怎么和 docker 及 rkt 引擎做交互的吧。定位代码的方式也很简单，直接选择 1.5 之前的版本，比如 v1.4.0 的 tag 版本。然后，既然官方说代码嵌在了 kubelet 代码里，我们可以直接切到[ kubelet 代码的目录](https://github.com/kubernetes/kubernetes/blob/v1.4.0/pkg/kubelet)，不难找到下面这两个子目录：

* [rkt](https://github.com/kubernetes/kubernetes/tree/v1.4.0/pkg/kubelet/rkt)

* [dockertools](https://github.com/kubernetes/kubernetes/tree/v1.4.0/pkg/kubelet/dockertools)

*注：这里还有一个小彩蛋，我们在该目录下还找到了一个[ rktshim ](https://github.com/kubernetes/kubernetes/tree/v1.4.0/pkg/kubelet/rktshim)的目录，说明当初各个容器运行时尚未普及对 CRI 的支持时，在 k8s 代码里嵌 xxxshim 服务用作临时支持是一个常规操作。*

> 那么，它们到底是咋交互的呢？

我们不难在 dockertools 目录下的 [kube_docker_client.go](https://github.com/kubernetes/kubernetes/blob/v1.4.0/pkg/kubelet/dockertools/kube_docker_client.go) 里面找到一个[ kubeDockerClient ](https://github.com/kubernetes/kubernetes/blob/v1.4.0/pkg/kubelet/dockertools/kube_docker_client.go#L38)的实现：

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

上面的注释也写的挺详细，大致意思就是它是一个和 Docker 交互的 client，并封装了一些 k8s 操作 Docker 需要的一些接口方法，这套接口方法具体定义在同一目录的 [docker.go](https://github.com/kubernetes/kubernetes/blob/v1.4.0/pkg/kubelet/dockertools/docker.go#L64) 里：

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

到这里，大致概括一下 k8s 在引入 CRI 阶段的一个迭代过程吧：

1、在 CRI 标准落地之前，k8s 等于是为每一个容器运行时都实现了一个具体的对接，rkt 和 dockertools 目录下即对应的代码实现；

2、官方于 1.5 版本开始正式引入 CRI 标准，并实现了对应的 shim 代码，如 dockershim 和 rktshim，在各个容器运行时尚未支持 CRI 标准的接口之前，充当一个胶水服务。

# 今生

通过追溯之前的版本历史，笔者终于了解了 k8s 在支持容器运行时这块的"坎坷经历"。

然而，最开始的问题始终未能得到解答：**为什么非得要弃用 dockershim ? 继续维护下去的话究竟会有哪些具体的痛点呢？**

毕竟，如果弃用 dockershim 的话，这意味着原本使用 docker 作为容器引擎的用户需要为此计划实施迁移到 containerd 或者其他支持 CRI 的容器运行时，这会是一个不小的时间和人力成本。

想要解答这个问题，恐怕还得先看看 dockershim 目前的使用场景以及 CRI 的发展现状。

## 启动前还要运行 dockershim 服务?

时至今日，kubelet 要去启动一个 docker 容器的话，究竟是怎么和 dockershim 配合工作的呢？不妨再来看看 kubelet 这层的代码实现。

这次笔者选的是刚发布不久的 [v1.22](https://github.com/kubernetes/kubernetes/tree/v1.22.0) 版本的代码。

kubelet 的启动入口位于 [cmd/kubelet/kubelet.go](https://github.com/kubernetes/kubernetes/blob/v1.22.0/cmd/kubelet/kubelet.go#L36)，熟悉 [cobra](https://github.com/spf13/cobra) 的朋友应该知道，它最终是会调用具体 Command 的 Run 方法。对于 kubelet 来说，调用的即是它实现的 [Run](https://github.com/kubernetes/kubernetes/blob/v1.22.0/cmd/kubelet/app/server.go#L155) 方法。

在经过一系列的处理后，kubelet 会走到核心的用来启动服务的 [run](https://github.com/kubernetes/kubernetes/blob/v1.22.0/cmd/kubelet/app/server.go#L514) 方法。直接看和 Dockershim 相关的部分！划到靠近函数末尾的部分，可以看到在真正启动前，kubelet 执行了一个 [kubelet.PreInitRuntimeService](https://github.com/kubernetes/kubernetes/blob/v1.22.0/cmd/kubelet/app/server.go#L796) 的操作。

> 这个 PreInitRuntimeService 方法做了什么事情呢？

不妨继续深入一下，看看它的[具体内容](https://github.com/kubernetes/kubernetes/blob/v1.22.0/pkg/kubelet/kubelet.go#L297)：

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

可以看到，当 `containerRuntime` 参数是 `kubetypes.DockerContainerRuntime` 时，kubelet 需要执行额外的 `runDockershim` 方法去启动一个 `dockershim` 服务（可以看到，上面有一行警告 `dockershim` 已弃用的提醒），而如果是 `kubetypes.RemoteContainerRuntime` 类型的话，则什么事情也不用干。

笔者还在 kubelet 目录下找到了 `kubelet_dockershim.go`，该文件里即实现了这个 [runDockershim](https://github.com/kubernetes/kubernetes/blob/v1.22.0/pkg/kubelet/kubelet_dockershim.go#L30) 方法，它会去调用 dockershim 的相关服务代码并启动一个 `dockerServer`。

很显然， kubelet 是通过这个 `dockershim` 服务包装的一层 CRI 接口调用 docker 启动 Pod 容器的。我们不妨看下 kubelet 实际是怎么去起 Pod 的，然后再来看看它是如何调用的容器运行时。

## kubeGenericRuntimeManager 的用途

回到 [cmd/kubelet/app/server.go](https://github.com/kubernetes/kubernetes/blob/v1.22.0/cmd/kubelet/app/server.go)，在执行了 `PreInitRuntimeService` 之后，不难发现 kubelet 会去执行 [RunKubelet](https://github.com/kubernetes/kubernetes/blob/v1.22.0/cmd/kubelet/app/server.go#L1108)，并最终通过 [kubelet.NewMainKubelet](https://github.com/kubernetes/kubernetes/blob/v1.22.0/cmd/kubelet/app/server.go#L1269) 来初始化 kubelet 服务实例。

*注：关于 kubelet 完整的启动逻辑，有位网易的同学写了一个[系列文章](https://mp.weixin.qq.com/s/g3C0alyd21fNhbj4OqPprQ)，有兴趣的朋友可以看看。*

这里面有关 runtime 部分最重要的就是[这一段](https://github.com/kubernetes/kubernetes/blob/v1.22.0/pkg/kubelet/kubelet.go#L662)了：

```
runtime, err := kuberuntime.NewKubeGenericRuntimeManager(
    ...
)
```

这里初始化了一个 `kubeGenericRuntimeManager` 的对象，它可以做哪些事情呢？我们暂且按下不表，先从 kubelet 这一层找找入口。回过头来，我们再来看看 kubelet 启动入口 `NewMainKubelet` 这块。可以看到，在初始化 `kubeGenericRuntimeManager` 之前，kubelet 初始化了一个 workQueue，并且初始化了一批 podWorker：

```
klet.podWorkers = newPodWorkers(
    klet.syncPod,
    klet.syncTerminatingPod,
    klet.syncTerminatedPod,

    kubeDeps.Recorder,
    klet.workQueue,
    klet.resyncInterval,
    backOffPeriod,
    klet.podCache,
)
```

熟悉 k8s 异步调谐这套控制器逻辑的朋友，应该能猜到。没错，这个 `podWorker` 就是监听 kubelet 关注的 Pod 资源的变化，并执行相应的调谐逻辑。这里先看一下 `syncPod` 这块的实现。

*注：有兴趣的朋友可以看看 `syncPod` 方法的[注释部分](https://github.com/kubernetes/kubernetes/blob/v1.22.0/pkg/kubelet/kubelet.go#L1498)，里面描述了 syncPod 的整体流程。*

syncPod 方法里的其他细节部分忽略，我们直接关注最终调用容器运行时服务同步 Pod 的[操作部分](https://github.com/kubernetes/kubernetes/blob/v1.22.0/pkg/kubelet/kubelet.go#L1729)：

```
result := kl.containerRuntime.SyncPod(pod, podStatus, pullSecrets, kl.backOff)
```

可以看到，这里 kubelet 实例调用的 [containerRuntime](https://github.com/kubernetes/kubernetes/blob/v1.22.0/pkg/kubelet/kubelet.go#L695) 毫无疑问便是之前 kubelet 在 `NewMainKubelet` 初始化 `kubeGenericRuntimeManager` 时创建出来的 `runtime` 实例：

```
runtime, err := kuberuntime.NewKubeGenericRuntimeManager(
    ...
)
klet.containerRuntime = runtime
```

那么，这个 runtime manager 具体又是怎么调用容器运行时服务来 SyncPod 的呢？

## 调用 runtime service 来 SyncPod

我们不妨先来看看 SyncPod 方法的注释部分：

```
// SyncPod syncs the running pod into the desired pod by executing following steps:
//
//  1. Compute sandbox and container changes.
//  2. Kill pod sandbox if necessary.
//  3. Kill any containers that should not be running.
//  4. Create sandbox if necessary.
//  5. Create ephemeral containers.
//  6. Create init containers.
//  7. Create normal containers.
func (m *kubeGenericRuntimeManager) SyncPod(pod *v1.Pod, podStatus *kubecontainer.PodStatus, pullSecrets []v1.Secret, backOff *flowcontrol.Backoff) (result kubecontainer.PodSyncResult) {
	...
}
```

可以看到，这就是一次经典的调谐逻辑。

按照它的说法，它会计算 Pod 当前的状态，然后按需清理环境，并尝试保证 Pod Sandbox 及相关容器（依次是 ephemeral container、init container 以及应用容器）处于运行状态。

快速浏览了一下 `SyncPod` 具体实现之后，不难发现，它将一些具体的实现部分放到了几个单独的方法里，如：[createSandbox](https://github.com/kubernetes/kubernetes/blob/v1.22.0/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L802)、[startContainer](https://github.com/kubernetes/kubernetes/blob/v1.22.0/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L884)。

这里，以 `createSandbox` 为例，看看 kubelet 在创建 Pod Sandbox 这块，调用 dockershim 和其他支持 CRI 的容器运行时有什么不同。略过生成 Pod 配置等步骤，直接看最核心的这一段：

```
podSandBoxID, err := m.runtimeService.RunPodSandbox(podSandboxConfig, runtimeHandler)
```

隐约可以猜到，这个 runtimeService 应该就是一个统一实现调用 CRI 的入口，不妨回过头来再看看 `kuberuntime.NewKubeGenericRuntimeManager` 这一步是怎么初始化这个 `runtimeService` 的：

```
runtime, err := kuberuntime.NewKubeGenericRuntimeManager(
	...
	kubeDeps.RemoteRuntimeService,
	...
)
```

咦？这个 `kubeDeps` 又是何方神圣呢？顺着源头找，可以看到它是 `NewMainKubelet` 就传入进来的一个参数项。再顺着调用链的源头，笔者找到了 [cmd/kubelet/app/server.go](https://github.com/kubernetes/kubernetes/blob/v1.22.0/cmd/kubelet/app/server.go) 里的 `RunKubelet`：

```
if err := RunKubelet(s, kubeDeps, s.RunOnce); err != nil {
	return err
}
```

再往上走便可以发现，这个 `kubeDeps` 早在 kubelet `NewKubeletCommand` 时候就已经[做了初始化](https://github.com/kubernetes/kubernetes/blob/v1.22.0/cmd/kubelet/app/server.go#L269)：

```
// use kubeletServer to construct the default KubeletDeps
kubeletDeps, err := UnsecuredDependencies(kubeletServer, utilfeature.DefaultFeatureGate)
if err != nil {
	klog.ErrorS(err, "Failed to construct kubelet dependencies")
	os.Exit(1)
}
```

但是仔细一看，里面并没有初始化 `RemoteRuntimeService` 啊，那什么时候做的呢？

啊！前文提到过，在执行 `RunKubelet` 前，kubelet 事先执行了 `PreInitRuntimeService`，它在里面是[这样](https://github.com/kubernetes/kubernetes/blob/v1.22.0/pkg/kubelet/kubelet.go#L334)初始化 `kubeDeps` 的相关运行时依赖的：

```
if kubeDeps.RemoteRuntimeService, err = remote.NewRemoteRuntimeService(remoteRuntimeEndpoint, kubeCfg.RuntimeRequestTimeout.Duration); err != nil {
	return err
}
```

想必这个 [pkg/kubelet/cri/remote/remote_runtime.go](https://github.com/kubernetes/kubernetes/blob/v1.22.0/pkg/kubelet/cri/remote/remote_runtime.go) 便是统一实现了调用 CRI 的 client 接口！

至此，kubelet 调用容器运行时的流程基本浮出了水面：

1、kubelet 在 `NewKubeletCommand` 命令入口便初始化了 `kubeDeps` 对象，用来存放一些 kubelet 需要的依赖；

2、在 Kubelet 执行 `RunKubelet` 之前它会先执行 `PreInitRuntimeService` 根据 `containerRuntime` 参数初始化 `runtimeService` 句柄并存放到 `kubeDeps` 便于后面部分调用；

3、在上一步骤中，如果是 docker 的话，会额外执行 `runDockershim` 启动 dockershim 服务；

4、执行 `RunKubelet` 方法时，它会进一步去执行 `NewMainKubelet` 并最终启动 kubelet 服务；

5、在 `NewMainKubelet` 这一步 kubelet 会初始化 Pod Worker 去执行 Pod 调谐，具体执行方法为 `syncPod`、`syncTerminatingPod` 等；

6、此外，`NewMainKubelet` 这一步还在初始化 `KubeGenericRuntimeManager` 的时候传入了 `kubeDeps.RemoteRuntimeService`，然后将 runtime manager 该实例赋给了 `kubelet.containerRuntime`；

7、当 kubelet 的 pod worker 进入主要的 syncPod 调谐周期时，它会调用 runtime manager 的 `SyncPod` 方法去做同步；

8、runtime manager 的 `SyncPod` 方法会做一系列判断，并执行相应的必要操作，比如 `createSandbox`，它会通过之前传入的 runtimeService 的 `RunPodSandbox` 方法调用具体的容器运行时服务做对应的事情。

## dockershim 的 CRI 实现

嗯 ，大致了解了 kubelet 调用容器运行时做 syncPod 调谐的这个过程了。那 dockershim 又是怎样具体实现这一套运行时接口的呢？

以 `RunSandbox` 这个接口为例，可以看到 dockershim 的实现里[做了大量手动操作的事情](https://github.com/kubernetes/kubernetes/blob/v1.22.0/pkg/kubelet/dockershim/docker_sandbox.go#L89)：

```
// RunPodSandbox creates and starts a pod-level sandbox. Runtimes should ensure
// the sandbox is in ready state.
// For docker, PodSandbox is implemented by a container holding the network
// namespace for the pod.
// Note: docker doesn't use LogDirectory (yet).
func (ds *dockerService) RunPodSandbox(ctx context.Context, r *runtimeapi.RunPodSandboxRequest) (*runtimeapi.RunPodSandboxResponse, error) {
	...
	// dockershim 会先保证 sandbox 镜像的存在，按需执行 docker pull
	if err := ensureSandboxImageExists(ds.client, image); err != nil {
		return nil, err
	}
	...
	// dockershim 还会根据配置手动创建 infra 容器
	createConfig, err := ds.makeSandboxDockerConfig(config, image)
	if err != nil {
		return nil, fmt.Errorf("failed to make sandbox docker config for pod %q: %v", config.Metadata.Name, err)
	}
	createResp, err := ds.client.CreateContainer(*createConfig)
	if err != nil {
		createResp, err = recoverFromCreationConflictIfNeeded(ds.client, *createConfig, err)
	}
	...
	// dockershim 手动创建 checkpoint
	if err = ds.checkpointManager.CreateCheckpoint(createResp.ID, constructPodSandboxCheckpoint(config)); err != nil {
		return nil, err
	}
	...
	// dockershim 调用 docker client 去启动容器
	// 注意，这个时候 infra 容器的网络栈还没设置
	err = ds.client.StartContainer(createResp.ID)
	if err != nil {
		return nil, fmt.Errorf("failed to start sandbox container for pod %q: %v", config.Metadata.Name, err)
	}
	...
	// 如果 dns 配置需要定制，dockershim 还会去手动重写该容器的 dns 配置
	// 这块是真的没想到，`rewriteResolvFile` 里就是一些调用操作系统接口去重写文件
	// docker client 难道没有提供设置 dns 的方式吗？
	...
		if err := rewriteResolvFile(containerInfo.ResolvConfPath, dnsConfig.Servers, dnsConfig.Searches, dnsConfig.Options); err != nil {
			return nil, fmt.Errorf("rewrite resolv.conf failed for pod %q: %v", config.Metadata.Name, err)
		}
	...
	// 为了能够调用 CNI 插件设置 infra 容器的网络栈
	// dockershim 还专门实现了一个 network 部分，它会给 CNI 插件传入相应的参数，设置 infra 容器的网络栈
	err = ds.network.SetUpPod(config.GetMetadata().Namespace, config.GetMetadata().Name, cID, config.Annotations, networkOptions)
	...
}
```

笔者在上述代码里添加了一些自己的注释。可以看到，k8s 的 kubelet 为了兼容支持 docker 容器运行时，做了大量胶水性质的粘合操作，比如设置 DNS Server 这种甚至是直接调用操作系统接口，以重写 resolv.conf 文件形式实现的！

*注1：dns 配置这块为什么是直接重写文件呢？为了解答这个问题，笔者找到了[最初实现版本](https://github.com/kubernetes/kubernetes/commit/5960d87d2142055cd29ebbce0243652c4adc5742#diff-40b456472817aeb853ac82dfc7cdf7632243c09bd40a085b74c5748580f6e104R237)，这里面是没有做任何重写操作。继续回溯历史，可以找到这个 [PR #43368](https://github.com/kubernetes/kubernetes/pull/43368)，似乎 dockertool 时代就已经是这种方式设置 DNS 了，为了支持 k8s 的一些 DNS 设置方面的功能，社区沿用了之前 dockertool 的方案，在 dockershim 处理 Pod Sandbox 的时候也加入了重写 resolv.conf 的逻辑。那么，为什么 dockertool 会重写 resolv.conf 呢，继续回溯版本后，笔者发现了关于 dns 设置这块的一段[注释](https://github.com/kubernetes/kubernetes/blob/v0.21.4/pkg/kubelet/dockertools/manager.go#L1235)，它的出处是 [PR 10266](https://github.com/kubernetes/kubernetes/pull/10266)。终于破案了，由于当时 docker 还不支持 ndots 选项，k8s 选择的是 hack 掉 infra 容器的 resolv.conf 来解决这个问题。*

*注2：接着上面一个注解，PR #10266 的确是通过魔改的方式给 k8s 加上了 ndots 选项的支持，但是，k8s 官方的核心开发人员 [thockin](https://github.com/thockin) 在同一年（ 2015 年）的九月份就给 docker 提了 PR（见 [PR #16031](https://github.com/moby/moby/pull/16031) ）加上了该功能。其实从这个事情也可以看出来，两个社区之间信息是不同步的，继续维护 dockershim 的话这样的问题还会不少。最好的解决办法恐怕还是将这些运行时方面的功能通过 CRI 标准接口定义好，然后容器运行时各自去实现。*

## containerd beyond 1.0

了解了 kubelet 调用 dockershim 这块的情况以后，笔者又想到了它的表兄弟 containerd，按道理它应该是 k8s 更为亲和的方案。那么，它在这个过程中扮演什么样的角色呢，现状又如何呢？

带着这个疑问，笔者克隆了 [containerd](https://github.com/containerd/containerd) 的仓库代码。通过 git log 很快便翻到了 commit 树的[起点](https://github.com/containerd/containerd/tree/15a96783ca2ac8c0eb2c400701e8eb335059c63b/)：

```
commit 15a96783ca2ac8c0eb2c400701e8eb335059c63b (HEAD)
Author: Michael Crosby <crosbymichael@gmail.com>
Date:   Thu Nov 5 15:29:53 2015 -0800

    Initial commit
```

可以看到，containerd 作为一个单独项目开发已经是 2015 年底了。有兴趣的朋友还可以翻阅一下这个起点 commit 的内容，其实等于就是从头开始写了...

> 那么，docker 什么时候开始集成 containerd 作为它的容器运行时呢？

其实也很简单，查一下 docker 仓库的 PR 历史就知道了。最终，笔者找到了 [PR #20662](https://github.com/moby/moby/pull/20662)。在这个 PR 变更内容里，很容易就找到了集成的 containerd 的版本：

```
ENV CONTAINERD_COMMIT 7146b01a3d7aaa146414cdfb0a6c96cfba5d9091
```

对比 commit 提交时间，大致是 [v0.1.0](https://github.com/containerd/containerd/tree/v0.1.0) 版本发布的时间。

在 containerd 单独立项开发的两年以后，2017 年 12 月份，[containerd 1.0 GA 了](https://www.cncf.io/blog/2017/12/05/general-availability-containerd-1-0/)，containerd 的核心开发人员 Michel Crosby 也[撰文](https://www.docker.com/blog/containerd-ga-features-2/)讲述了 containerd 抵达 1.0 的这个旅程，其中包括像从 Graphdriver 切换到 Snapshot 这样的架构层面的重新设计。

而在此之前的 11 月份，k8s 1.8 加入了对 containerd 运行时的支持，见 [1.8 changelog](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.8.md#container-runtime-interface-cri)。

*注1：有趣的是，containerd 自己也引入了一个 containerd-shim，这个 shim 是为了让出自 containerd 的容器进程能够和 containerd 解耦，具体见 containerd v0.5 的 [PR #98 title](https://github.com/containerd/containerd/pull/98#issue-58078723)。*

*注2：此外，值得一提的是，引入 containerd 后的 docker 自身也不是太稳定（当然，剥离 containerd 之前笔者在生产环境使用 docker daemon 也遇到过不少问题），笔者自己就经历过一个诡异问题，具体可以参考笔者 17 年时候写的[这篇博客](https://colstuwjx.github.io/2017/06/%E8%AE%B0%E4%B8%80%E6%AC%A1%E5%A4%B1%E8%B4%A5%E7%9A%84docker%E6%8E%92%E9%9A%9C%E7%BB%8F%E5%8E%86/)，现在回过头来看，可能和 containerd-shim 的这个玩法有关系。顺便说一句，那会儿的 containerd 尽管已经 1.0 了，UX 交互却还是相当简陋，这也是很多用户在 containerd 可以单独作为容器运行时选项时仍然坚持选择 docker 的重要原因之一。有兴趣的朋友可以看下笔者在 18 年初试玩 containerd 的[经历](https://colstuwjx.github.io/2018/02/%E5%8E%9F%E5%88%9B-%E5%B0%8F%E5%B0%9Dcontainerd%E4%B8%80/)。*

## 从 docker 到 containerd 的迁徙

时至今日，CRI 已然在各个主流的容器运行时得到支持和普及，containerd 的一些周边支持也逐渐完善起来，比如命令行工具这块，`crictl` 沿用了之前 `docker` 留下来的操作习惯，相关命令均可以接近无缝地切换到 `crictl` 。

业内也出现一些从 docker 引擎迁移到 containerd 的案例，如 eBay 早在 2019 年就将运行时[从 docker 切换到了 containerd](https://www.infoq.cn/article/odslclsjvo8bnx*mbrbk)，各大公有云提供的 Kubernetes 服务也在 k8s 官方宣布弃用 dockershim 支持后不久便宣布[使用 containerd 替换 docker](https://thenewstack.io/azure-kubernetes-service-replaces-docker-with-containerd/)。

# 结语

呼，花了点时间，终于摸清了 dockershim 的身世背景。整体看下来，似乎和 k8s 官方博客里说的差不多。笔者也感受到，在迭代过程中社区的开发人员为了弥补 k8s 和 docker 之间的 gap 做出的一些妥协：比如前面提到的实现 dockershim 让 docker 支持 CRI 标准，以及重写 resolv.conf 来支持 k8s 的一些 dns 功能等等。

出于开发和运维方面的复杂性考虑，无论是 k8s 官方弃用 dockershim 还是社区用户将运行时切换到 containerd 其实都是非常理性的做法。

只是，似乎 docker 的那个时代已经落幕了。

# 参考

1、https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#deprecation

2、https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/

3、https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/

4、https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md#design-docs-and-proposals

5、https://opencontainers.org/about/overview/

6、https://dev.to/inductor/wait-docker-is-deprecated-in-kubernetes-now-what-do-i-do-e4m
