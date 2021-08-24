---
title: "[ 原创 ] containerd 迁移二三事"
date: 2021-08-23T08:00:00+08:00
tags: ["cloud-native", "containerd", "docker"]
categories: ["cloud-native"]
comments: false
showMeta: false
showAction: false
aliases:
    - /migration-to-containerd/
---

之前有写过一篇[试玩 containerd ](https://colstuwjx.github.io/tasting-containerd-01/)的文章，一晃已经3年多了，业内已经有不少人选择抛弃 docker，直接换成 containerd 了。

那么，这玩意儿到底好使不，切换过程中又会有哪些问题呢？今天笔者就来分享一下最近从 docker 迁移到 containerd 做的一些工作，以及个人在这个过程中的一些思考。

<!--more-->

## 背景

首先介绍一下这件事情的背景。我们内部有运行多套 k8s 集群，为了消费 v1.20 版本引入的一些新特性，我们打算升级一下集群的版本。然而，一旦升级到 v1.20 ，官方已经明确表示将会逐渐弃用对 `dockershim` 的支持。要不要把运行时也一起切掉呢？一番讨论过后，我们最终还是决定趁着这次升级集群的机会，把运行时也切换到 `containerd` 。

*注：关于弃用 dockershim 这块，笔者也专门写了一篇[文章](https://colstuwjx.github.io/dive-into-sourcecode-why-k8s-deprecated-dockershim/)分析代码层面维护 dockershim 的成本，有兴趣的朋友可以看看*

## 几个问题

### 变更配置：启用 cri plugin

我们内部是用 [Kubespray](https://github.com/kubernetes-sigs/kubespray) 这套 Ansible 脚本来管理 k8s 集群的。从 docker 切换到 containerd 似乎只是改一行 playbook 变量的问题。

然而，实际当然没有这么简单。首先遇到的第一个问题便是配置的问题：docker 此前的部署默认会禁用 containerd 有关 CRI 的 plugin 。切换到直接使用 containerd 以后，我们需要手动修改 containerd 的相关配置：

```
$ vim /etc/containerd/config.toml
#disabled_plugins = ["cri"]
```

此外，我们在测试环境把 docker 切换到 containerd 以后发现之前的 docker 容器还有一些残留，这也需要运维手动清理。

*注：如何将运行时自动安全地从 docker 切换到 containerd，可以参考 eBay 分享的这篇文章：https://www.infoq.cn/article/odslclsjvo8bnx\*mbrbk*

### 监控适配：cadvisor

整个切换的过程除了中间的一点小插曲倒还算顺利，跑了几天发现似乎也挺稳定的（毕竟 docker 也是用的这个 runtime ）。这是不是就完事了呢？等一下，咦，监控好像没数据了！什么情况？

原来，我们当初落地 k8s 的时候还搞了几个配套的周边组件，用来解决监控日志之类的需求，其中就有用到 cadvisor 来采集容器层面的基础指标，比如 CPU、内存使用率等等。

为了区分每个业务应用的数据，便于做单独的监控和告警，笔者在 cadvisor 这一层也做了一些改动，详细信息见 [PR #2330](https://github.com/google/cadvisor/pull/2330)，主要就是设置了一个容器 env 白名单，然后把这些 env 变量作为各项监控指标的 label 对外提供，这样我们便可以传入一些标识应用的环境变量，比如 `APP_ID`，方便做应用维度的监控告警。

但是，从这个 PR 的[评论部分](https://github.com/google/cadvisor/pull/2330#issuecomment-545656563)也可以看到，社区开发人员指出，把容器的 env 作为标签加到 cadvisor 对外暴露的 `/metrics` 接口的话，不只是 docker，containerd 和其他运行时也都会受影响。因此，他建议把 `docker_env_metadata_whitelist` 这个参数改成统一的 `env_metadata_whitelist`，这样所有容器运行时都可以消费这个 flag。

当时因为时间精力有限，笔者选择的是暂时先搁置这个 PR，手动在自己 fork 的 repo 里 merge 了[该 commit ](https://github.com/Colstuwjx/cadvisor/commit/96c96bb801a85bdf3b2f88a605a9053a1f95806d)，随后在我们内网环境使用这份修改后的版本。然而时隔将近两年后，因为切换到 containerd 的缘故，终究还是得把这件事情做完。

与此同时，笔者又想到另外一个问题，cadvisor 对于 containerd 的支持力度到底怎么样？

> 第一个问题就是容器启停这块的事件处理机制，cadvisor 一般是作为一个 DaemonSet 运行在宿主机上，这就不可避免地遇到一个问题：怎么知道自己监控的对象？

查看代码后不难发现，cadvisor 是有实现一套统一的 [ContainerWatcher](https://github.com/google/cadvisor/blob/v0.40.0/watcher/watcher.go#L45) 的接口：

```
type ContainerWatcher interface {
	// Registers a channel to listen for events affecting subcontainers (recursively).
	Start(events chan ContainerEvent) error

	// Stops watching for subcontainer changes.
	Stop() error
}
```

它似乎约定了每个容器运行时得自己去实现各自的 watch 机制，那么，是否真的是这样呢？我们不妨接着看下去：

```
...
// Listen to events from the container handler.
go func() {
    for {
        select {
        case event := <-m.eventsChannel:
            switch {
            case event.EventType == watcher.ContainerAdd:
                switch event.WatchSource {
                default:
                    // 监听新加容器的事件并响应
                    err = m.createContainer(event.Name, event.WatchSource)
                }
            case event.EventType == watcher.ContainerDelete:
                // 监听删除容器的事件并响应
                err = m.destroyContainer(event.Name)
            }
            if err != nil {
                klog.Warningf("Failed to process watch event %+v: %v", event, err)
            }
        case <-quit:
            // 停止之前启动的各个 watcher，然后平滑退出
            ...
        }
    }
}()
...
```

如上，笔者在 cadvisor 业务逻辑的入口实现 [manager.go](https://github.com/google/cadvisor/blob/v0.40.0/manager/manager.go) 里面找到了相关事件的[处理循环](https://github.com/google/cadvisor/blob/v0.40.0/manager/manager.go#L1168)，见上面笔者添加的一些注释，似乎就是中规中矩的监听各个添加或删除的事件然后执行相应的操作。那么，我们不妨看看各个容器运行时具体的 handler 是怎么实现的吧！

```
...
container.RegisterContainerHandlerFactory(f, []watcher.ContainerWatchSource{watcher.Raw})
...
```

咦？无论是 [docker](https://github.com/google/cadvisor/blob/v0.40.0/container/docker/factory.go#L388)，还是 [containerd](https://github.com/google/cadvisor/blob/v0.40.0/container/docker/factory.go#L388)，又或者是 [cri-o](https://github.com/google/cadvisor/blob/v0.40.0/container/crio/factory.go#L160)，全都是注册的 Raw 这个 watcher，根本没有去实现各自的 `ContainerWatcher`。

且继续看下去。笔者再次找到 manager 这块的代码，可以看到，各个容器运行时貌似是有机会去注册单独的 `watcher`：

```
// Start the container manager.
func (m *manager) Start() error {
    // 各个容器运行时自己提供的 watcher 的初始化
	m.containerWatchers = container.InitializePlugins(m, m.fsInfo, m.includedMetrics)

    ...

    rawWatcher, err := raw.NewRawContainerWatcher()
    if err != nil {
        return err
    }
    // 单独追加的 raw watcher
    m.containerWatchers = append(m.containerWatchers, rawWatcher)

    ...
}
```

但是，翻到具体运行时代码的时候就会发现，它压根就没实现，下面即是 docker 注册自己的时候实现的 `Register` 方法：

```
func (p *plugin) Register(factory info.MachineInfoFactory, fsInfo fs.FsInfo, includedMetrics container.MetricSet) (watcher.ContainerWatcher, error) {
	err := Register(factory, fsInfo, includedMetrics)
	return nil, err
}
```

哈哈，直接返回的 `nil`！那它怎么拿到容器的生命周期，然后做监控采集的处理呢？答案就在于 manager 单独加进去的这个 raw watcher。

我们不妨一起来看看这个 raw watcher 具体的代码实现。watcher 最开始会初始化一个 inotify 的 watcher，然后会给出对应的 cgroup 挂载路径：

```
func NewRawContainerWatcher() (watcher.ContainerWatcher, error) {
    ...
	watcher, err := common.NewInotifyWatcher()
	if err != nil {
		return nil, err
	}

	rawWatcher := &rawContainerWatcher{
		cgroupPaths:      common.MakeCgroupPaths(cgroupSubsystems.MountPoints, "/"),
		cgroupSubsystems: &cgroupSubsystems,
		watcher:          watcher,
		stopWatcher:      make(chan error),
	}

	return rawWatcher, nil
}
```

在 watcher 启动时，watcher 会去遍历指定的 cgroup 目录，然后注册对应的监听器，事件源即是之前初始化的 inotify watcher 监听到目录变化后发出的事件：

```
func (w *rawContainerWatcher) Start(events chan watcher.ContainerEvent) error {
	// Watch this container (all its cgroups) and all subdirectories.
	watched := make([]string, 0)
    // watch 给定的 cgroup 路径，然后维护一组监听器
	for _, cgroupPath := range w.cgroupPaths {
		_, err := w.watchDirectory(events, cgroupPath, "/")
		if err != nil {
			for _, watchedCgroupPath := range watched {
				_, removeErr := w.watcher.RemoveWatch("/", watchedCgroupPath)
				if removeErr != nil {
					klog.Warningf("Failed to remove inotify watch for %q with error: %v", watchedCgroupPath, removeErr)
				}
			}
			return err
		}
		watched = append(watched, cgroupPath)
	}

	// Process the events received from the kernel.
	go func() {
		for {
			select {
			case event := <-w.watcher.Event():
                // 处理获取到的事件
				err := w.processEvent(event, events)
				if err != nil {
					klog.Warningf("Error while processing event (%+v): %v", event, err)
				}
            ...
            }
        }
    }()
    ...
}
```

为什么会用 cgroup + inotify 这套机制呢？其实仔细想想也挺合理，cadvisor 开发的时候 CRI 标准都还没有呢，要是指望运行时这层给出一套统一的事件接口再来开发，那黄花菜都凉了。

> 除此之外，还有一个问题就是，cadvisor 怎么采集 containerd 的 metrics 呢？

翻阅代码之后一目了然：

```
func (h *containerdContainerHandler) GetStats() (*info.ContainerStats, error) {
    // 可以看到 contaienrd 是直接复用 libcontainer 的逻辑来采集监控数据
    // 当然，docker 也是通过这种方式做的，具体可以看下 https://github.com/google/cadvisor/blob/v0.40.0/container/docker/handler.go#L460
	stats, err := h.libcontainerHandler.GetStats()
	if err != nil {
		return stats, err
	}
	// Clean up stats for containers that don't have their own network - this
	// includes containers running in Kubernetes pods that use the network of the
	// infrastructure container. This stops metrics being reported multiple times
	// for each container in a pod.
	if !h.needNet() {
		stats.Network = info.NetworkStats{}
	}

	// Get filesystem stats.
	err = h.getFsStats(stats)
	return stats, err
}
```

正如注释说的，它[这里](https://github.com/google/cadvisor/blob/v0.40.0/container/containerd/handler.go#L196)是复用的 libcontainer 的逻辑。那这个 libcontainer 的 handler 具体又是怎么实现 `GetStats` 方法的呢？

```
// Get cgroup and networking stats of the specified container
func (h *Handler) GetStats() (*info.ContainerStats, error) {
    ...
    // 主要的监控数据，如 CPU、内存等是读取的 cgroup 数据
    cgroupStats, err := h.cgroupManager.GetStats()
    ...
    libcontainerStats := &libcontainer.Stats{
		CgroupStats: cgroupStats,
	}
    ...

	// If we know the pid then get network stats from /proc/<pid>/net/dev
	if h.pid > 0 {
		if h.includedMetrics.Has(container.NetworkUsageMetrics) {
            // 网络层面的相关监控数据是通过读取 proc 文件系统得到的
            netStats, err := networkStatsFromProc(h.rootFs, h.pid)
        }
        ...
    }
    ...
}
```

soga，看来主要是依赖 cgroup 还有读取 /proc 文件系统来汇总的监控数据。所以从 docker 切换到 containerd，cadvisor 在监控采集这块的具体实现上差别不大。

当然，前面提到的笔者自己做的一些修改，这次切换到 containerd 以后自然还是要做一下兼容的。做法也挺简单，就是按照社区开发人员建议的，合并这些 flag，然后统一做注入操作，这块笔者也给社区反馈了一个单独的 PR，见 [PR #2921](https://github.com/google/cadvisor/pull/2921)。

### 日志适配：log-pilot

搞定了监控这块以后，笔者又遇到了一个更棘手的问题：日志方案也不好使了。

之前搞日志采集方案的时候，笔者选用的是阿里云容器团队开源的 [log-pilot](https://github.com/AliyunContainerService/log-pilot) 项目。它的实现原理说来也挺简单的，就是嵌了一个日志采集器，然后同时监听 docker 的 event，读取相应容器的 volume 信息，再根据指定的 env 或者 label 标签，在自己的目录下生成对应的日志采集器（比如 filebeat 或是 fluentd ）的配置。

log-pilot 原本是监听的 docker event 来做事的，切换到 containerd 以后直接就给整罢工了。

那么，这个方案到底能不能用在 containerd 上面呢？核心即是要解决下面两个问题：

> 1. containerd 或者 CRI 能否也提供一套 event 机制供订阅消费？

> 2. 能否通过 containerd 或者 CRI 接口拿到容器的详细信息（最核心的如容器的 ID、logPath、env、volume mount 等）？

这些问题确实还蛮棘手的，笔者也是花了大概一周左右的时间详细了解了一下 containerd 和 cri-api 的相关细节，终于有所收获。

话不多说，直接上代码。

首先，为了解决第一个问题，我们先来看看 log-pilot 是如何消费 docker event 的：

```
...
msgs, errs := p.client.Events(ctx, options)

go func() {
    ...
    for {
        select {
        case msg := <-msgs:
            if err := p.processEvent(msg); err != nil {
                log.Errorf("fail to process event: %v,  %v", msg, err)
            }
        ...
    }
}()
```

可以看到，log-pilot 确实是调用 docker client 的 `Events` 方法获取事件信息，然后在拿到消息后执行对应的处理逻辑。那切换到 containerd 之后，怎么获取容器启停的这个事件源呢？翻遍[ CRI 接口](https://github.com/kubernetes/cri-api/blob/master/pkg/apis/services.go#L98)，笔者并没有找到任何有关 event 的接口：

```
// RuntimeService interface should be implemented by a container runtime.
// The methods should be thread-safe.
type RuntimeService interface {
	RuntimeVersioner
	ContainerManager
	PodSandboxManager
	ContainerStatsManager

	// UpdateRuntimeConfig updates runtime configuration if specified
	UpdateRuntimeConfig(runtimeConfig *runtimeapi.RuntimeConfig) error
	// Status returns the status of the runtime.
	Status() (*runtimeapi.RuntimeStatus, error)
}
```

那 containerd 本身有没有提供这方面的接口呢？谷歌搜索可以找到 containerd 仓库里这部分的代码 [events.go](https://github.com/containerd/containerd/blob/v1.5.5/events/events.go)：

```
// Subscriber allows callers to subscribe to events
type Subscriber interface {
	Subscribe(ctx context.Context, filters ...string) (ch <-chan *Envelope, errs <-chan error)
}
```

containerd 还真有提供事件订阅机制，那么问题来了，怎么用呢？很遗憾，这方面的文档非常稀缺，笔者不得不通过代码里的 [testcase](https://github.com/containerd/containerd/blob/v1.5.5/events/exchange/exchange_test.go) 来了解具体的调用方式：

```
func TestExchangeBasic(t *testing.T) {
	ctx := namespaces.WithNamespace(context.Background(), t.Name())
	testevents := []events.Event{
		&eventstypes.ContainerCreate{ID: "asdf"},
		&eventstypes.ContainerCreate{ID: "qwer"},
		&eventstypes.ContainerCreate{ID: "zxcv"},
	}
	exchange := NewExchange()

	t.Log("subscribe")
	var cancel1, cancel2 func()

	// Create two subscribers for same set of events and make sure they
	// traverse the exchange.
	ctx1, cancel1 := context.WithCancel(ctx)
	eventq1, errq1 := exchange.Subscribe(ctx1)
    ...
}
```

OK，event 这块有了，那么第二个问题怎么解决呢？如何获取容器的明细信息呢？

log-pilot 之前是通过调用 docker client 的 `ContainerList` 和 `ContainerInspect` 来获取容器的明细。那么，如今是否可以通过调用 containerd 对外暴露的 CRI 接口来实现这一步呢？如果可以的话，那这块代码便可以自然而然地推广到其他支持 CRI 接口的运行时了。

然而，很遗憾，答案是否定的。这一块也是笔者最困惑的地方，CRI 标准里的确是有列出所有容器和获取某个容器详细状态的[接口](https://github.com/kubernetes/cri-api/blob/v0.22.1/pkg/apis/services.go#L42)：

```
...
// ListContainers lists all containers by filters.
ListContainers(filter *runtimeapi.ContainerFilter) ([]*runtimeapi.Container, error)
// ContainerStatus returns the status of the container.
ContainerStatus(containerID string) (*runtimeapi.ContainerStatus, error)
...
```

然而，这上面接口返回的信息少了一些关键字段，比如 [runtimeapi.Container](https://github.com/kubernetes/cri-api/blob/v0.22.1/pkg/apis/runtime/v1alpha2/api.pb.go#L4141) 里面**根本就没有容器 env 和 volume 相关字段**，更让人迷惑的是，[CreateContainer](https://github.com/kubernetes-sigs/cri-tools/blob/v1.22.0/vendor/k8s.io/cri-api/pkg/apis/services.go#L35) 创建容器接口方法里却要求用户传入容器配置 [ContainerConfig](https://github.com/kubernetes/cri-api/blob/v0.22.1/pkg/apis/runtime/v1alpha2/api.pb.go#L3390)，里面就有 env 和 mount 相关的定义。

再者，[runtimeapi.ContainerStatus](https://github.com/kubernetes/cri-api/blob/v0.22.1/pkg/apis/runtime/v1alpha2/api.pb.go#L4366) 里面尽管有 mount 字段，却不是全部的 volume mount 数据，比如在 Dockerfile 里通过 `VOLUME` 指令创建的 image volume 就不包含在内。

事情似乎又走到了死胡同。

诶？等一下，平时敲 `crictl inspect <CONTAINER_ID>` 的时候是可以看到 env 和 volume 的，`crictl` 这个工具它是怎么拿到的呢？翻阅了 crictl 的[代码实现](https://github.com/kubernetes-sigs/cri-tools/blob/v1.22.0/cmd/crictl/container.go#L390)以后一下就恍然大悟了：

```
var containerStatusCommand = &cli.Command{
	Name:      "inspect",
	Usage:     "Display the status of one or more containers",
	ArgsUsage: "CONTAINER-ID [CONTAINER-ID...]",
	Action: func(context *cli.Context) error {
        ...
        runtimeClient, runtimeConn, err := getRuntimeClient(context)
        if err != nil {
            return err
        }
        defer closeConnection(context, runtimeConn)

        for i := 0; i < context.NArg(); i++ {
            containerID := context.Args().Get(i)
            err := ContainerStatus(runtimeClient, containerID, context.String("output"), context.String("template"), context.Bool("quiet"))
            if err != nil {
                return errors.Wrapf(err, "getting the status of the container %q", containerID)
            }
        }
        return nil
    },
}
```

可以看到，它实例化了一个 `runtimeClient`，然后获取 container 状态的时候用到了这个 client。那么不妨再来看看这个 client 是啥：

```
func getRuntimeClient(context *cli.Context) (pb.RuntimeServiceClient, *grpc.ClientConn, error) {
	// Set up a connection to the server.
	conn, err := getRuntimeClientConnection(context)
	if err != nil {
		return nil, nil, errors.Wrap(err, "connect")
	}
	runtimeClient := pb.NewRuntimeServiceClient(conn)
	return runtimeClient, conn, nil
}
```

明白了，它在[这里](https://github.com/kubernetes-sigs/cri-tools/blob/v1.22.0/cmd/crictl/util.go#L182)初始化了一个通过 grpc 连接到 containerd 的 client。这个和笔者之前了解到的 CRI 调用方式还有一些出入，笔者以为会是下面这样：

```
import (
    ...
    internalapi "k8s.io/cri-api/pkg/apis"
    "k8s.io/kubernetes/pkg/kubelet/cri/remote"
    ...
)
// 之前笔者以为 cri-tool 会通过类似这样的方式去初始化一个 CRI 标准的 client 服务
// internalapi.RuntimeService 即是所有对外暴露的 CRI 接口
func getRuntimeService(context *cli.Context) (internalapi.RuntimeService, error) {
	return remote.NewRemoteRuntimeService(RuntimeEndpoint, Timeout)
}
```

那么，cri-tool 为什么会选择用 grpc 的方式去调用 contaienrd 的接口，而不是标准的 CRI 呢？

对比之下，笔者发现，`internalapi.RuntimeService` 返回的信息就是标准的 CRI 接口约定的内容，而 grpc 版本的返回里会附带一些额外信息，比如在执行 `crictl inspect <CONTAINER_ID>` 的时候，它在[这里](https://github.com/kubernetes-sigs/cri-tools/blob/v1.22.0/cmd/crictl/container.go#L866)会将获取的一些额外信息也打印输出到屏幕：

```
return outputStatusInfo(status, r.Info, output, tmplStr)
```

这里的 [Info](https://github.com/kubernetes-sigs/cri-tools/blob/v1.22.0/vendor/k8s.io/cri-api/pkg/apis/runtime/v1alpha2/api.pb.go#L4550) 也即是调用 [ContainerStatus](https://github.com/kubernetes-sigs/cri-tools/blob/v1.22.0/vendor/k8s.io/cri-api/pkg/apis/runtime/v1alpha2/api.pb.go#L7781) 这个 grpc 接口拿到的 [ContainerStatusResponse](https://github.com/kubernetes-sigs/cri-tools/blob/v1.22.0/vendor/k8s.io/cri-api/pkg/apis/runtime/v1alpha2/api.pb.go#L4543) 里的其中一部分。

也就是说，要想拿到容器的 env 和完整的 volume mount 等这些"额外信息"，光靠 CRI 接口是不够用的（这感觉挺扯淡的）。

话说回来，要是想解决第二个问题，看上去只需要自己手动解析一下这个 `Info`，然后拿到对应的 env 和 volume 数据就行了。至此，基本都有解决办法了，可以开搞了！

通过少许的改造，笔者最终也是完成了这块的适配工作，详细改动见 [Colstuwjx/log-pilot PR #2](https://github.com/Colstuwjx/log-pilot/pull/2)。

在开发过程中，笔者也遇到了一些问题：

1. 不同于 docker 容器默认输出的 jsonlog，containerd 容器标准输出的日志格式是 plain text，log-pilot 生成的 filebeat、fluentd 配置需要做一些相应的适配；

2. containerd 置备的 volume 只是单纯作为卷挂载到容器里，并没有像 docker 那样作为一种资源显式管理起来，比如 docker 可以执行 `docker volume ls` 来查看对应的卷。因此，我们需要统一封装一层，不能再完全照搬之前 inspect volume 这块的逻辑；

3. 在切换到 containerd 以后，如果机器上同时还跑着 docker daemon ，再用 docker 去启动一个容器的话，containerd 会监听到一些 exit 类型的垃圾事件，怀疑可能是错误拿到了 docker 容器的相关事件。解决办法也很简单：就是一个宿主机上只跑一种 runtime；

4. 在实际运行改版后的 log-pilot 后，笔者发现时不时会收到一条 task start 事件，一路追查下来发现，该容器的 ID 竟然是某个 Pod sandbox ID，想必 containerd 是监听到了 Pod 的 sandbox 容器启动事件。这块其实也比较迷惑，按道理 containerd 应该不会再向用户展示 sandbox 容器这一层了，事实上，从命令也可以看到，sandbox 是通过一个单独的命令来查询的，然而在 event 这一层它们又变成对等的 container 了，挺奇怪的逻辑。最终，笔者通过在拿到事件后手动检查一下是否是 Pod Sandbox ID 的 workaround 方式变相解决了这个问题。

## 结语

这一番折腾下来总算搞定了，万万没想到最花时间精力的部分竟然是周边的监控和日志组件的适配。

其实这个过程中间，相信大家也能看出来，k8s 和 docker 两个开发阵营之间的 gap 还是很多的，这里列举几个比较典型的吧：

- cadvisor 在开发的时候压根就没考虑过调用 docker 接口之类的，直接读 cgroup 和 /proc，这个做法也一直延续至今；

- CRI 约定的只是 kubelet 管理容器相关需要的一些接口，至于 events、volume 这些都是没有包含在内的。其实个人认为至少应该把 image volume（即在 Dockerfile 里面通过 `VOLUME` 指定的匿名卷，它的定义会体现在 OCI 镜像里 ）和 container volume 这些给明确定义出来，而不是放任不管，然后在实现的时候一股脑塞到原生 grpc 接口的 extra info 里...

- 如果想用 crictl 或者类似的工具去调用 containerd 裸起一个容器，相信我，这真不是一件简单的事儿。其实也能理解，这并不是 k8s 社区关注的重点；

- containerd 原生暴露的 event 接口里能够监听到 sandbox 容器 start 的事件，然而在查询容器这层面的接口又无法查到 Sandbox 容器的信息，那么 Sandbox 到底是不是一个 container 呢？似乎和 dockershim 时代的 infra container 相比，Sandbox 这个概念在 containerd 具体实现时变得更加模糊了；

- containerd 自身是有区分 namespace 的，比如 docker 容器默认都在 `moby` 这个 namespace 下面，k8s 的则会是 `k8s.io`。此 namespace 非 k8s 的那个彼 namespace。这个 namespace 概念感觉有点类似于 jenkins 的 workspace 的意思，然而它是没有体现到 CRI 标准里的，这也就造成了我们实际在用 containerd 的时候，只有当找不到容器了，才会意识到有这个 namespace 的区分；

- 早期的云原生开发者多是站在 docker 用户的角度思考问题，这也是为什么 log-pilot 会直接选择把宿主机目录以只读形式挂载到它的容器里，这样来采集日志。然而，时代变了，今天回头再看 log-pilot 的这个做法，恐怕已经是有悖于云原生理念了，至少在安全性方面是很难达标的；

- docker 的 event 机制乍一看好像和 k8s 的 informer 机制差别不大，但是仔细一想其实出入挺大的：event 传递的是变化的增量信息，informer 除了传递这个变化以外，还鼓励和提倡遵循 "reconcile" 这套理念。

*注：当然，以上这些也只是笔者个人看到的一些情况，未必是完全准确的，欢迎拍砖。*

此外，这次迁移也给了笔者一次反思的机会。时移世易，有些在之前看起来理所当然的行为模式，如今回过头来看可能会有一些新的看法。

完。

## 参考

- https://www.infoq.cn/article/odslclsjvo8bnx*mbrbk

- https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md

- https://github.com/kubernetes-sigs/cri-tools/blob/master/cmd/crictl/container.go
