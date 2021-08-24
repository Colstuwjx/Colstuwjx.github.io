---
title: "[ 原创 ] 小尝 containerd（一）"
date: 2018-02-05T20:00:00+08:00
tags: ["cloud-native", "containerd", "docker"]
categories: ["cloud-native"]
comments: false
showMeta: false
showAction: false
aliases:
    - /tasting-containerd-01/
---

[containerd](https://containerd.io/)是一个开源的container runtime实现，也是docker背后管理容器生命周期的功臣，借着周末的一些时间，鄙人也小小把玩了一下containerd，有任何不对的地方欢迎指正。

<!--more-->

#### 架构

containerd和runc、docker的关系其实比较复杂，我们不妨简单地理解成，containerd是一个纯粹的，面向容器工业标准的一个容器管理运行时，runc是一个包装了libcontainer的满足OCI标准的类库，docker则更像是一个承载容器而又不想只做容器引擎的一个复杂产物。下图展示了containerd在容器生命周期里扮演的角色：

![layout](/images/2018/Feb/layout-1.png)

不难发现，containerd本身负责管理容器整个的生命周期，从镜像拉取，到挂载fs，再到启动，销毁等，它对外提供grpc形式的API，对内通过调用runc等类库来创建满足OCI标准的容器，并且屏蔽了底层的实现细节。

在架构设计和实现方面，官方的核心开发人员在他们的[博客](https://blog.docker.com/2017/12/containerd-ga-features-2/)里也提到了，通过[反思graphdriver的实现](https://blog.mobyproject.org/where-are-containerds-graph-drivers-145fc9b7255)，他们将containerd设计成了snapshotter的模式，这也使得containerd对于overlay文件系统/快照文件系统的支持均称得上是不错。其具体的架构实现如下：

![arch](/images/2018/Feb/architecture-1.png)

storage、metadata和runtime的三大块划分非常清晰，通过抽象出events的设计，containerd也得以将网络层面的复杂度交给了上层处理，仅提供对network ns的一些添加接口和配置等操作的API。这样做的好处无疑是巨大的，保留最小功能集合的纯粹和高效，而将更多的复杂性及灵活性交给了插件及上层系统。

#### 试玩

要运行containerd的话，我们需要从github下载containerd的release版本，这里鄙人测试使用的是[v1.0.1](https://github.com/containerd/containerd/releases/download/v1.0.1/containerd-1.0.1.linux-amd64.tar.gz)，

可以看到，里面有几个binary，它们分别实现了如下功能：

* containerd即容器的运行时，以GRPC的形式提供满足OCI标准的API；

* containerd-release和containerd-stress则分别是containerd项目的发行版发布工具以及压力测试工具，以后有时间再来八；

* ctr是一个简单的CLI接口，用作containerd本身的一些调试用途，投入生产使用时还是应该配合docker或者cri-containerd部署；

* containerd-shim是每一个容器的运行时载体，我们在docker宿主机上看到的shim也正是代表着一个个通过调用containerd启动的docker容器。

containerd需要以daemon方式运行在宿主机上，我们可以轻松地在github仓库里找到官方编写的[systemd service](https://github.com/containerd/containerd/blob/master/containerd.service)文件，而containerd的配置文件为`/etc/containerd/config.toml`，内容如下：

```
subreaper = true
oom_score = -999

[debug]
 level = “debug”

[metrics]
 address = “127.0.0.1:1338”

[plugins.linux]
 runtime = “runc”
 shim_debug = true
```

我们先卖个关子，在后面文章里讲解这里配置参数的含义。通过熟悉的`systemctl start containerd`，如下图所示，containerd启动了！

![containerd-service](/images/2018/Feb/containerd.jpg)

##### ctr cli

下面，我们不妨用官方提供的ctr这一简单的CLI工具小小试玩一下用containerd来运行容器。

```
# containerd支持的镜像格式有多种，这里我们不妨试一试用docker导出一个镜像tar，然后用runc配置oci标准所需的参数，然后再用ctr启动之，参考：https://github.com/projectatomic/containerd/blob/master/docs/bundle.md

# step 1. mkdir && pull image
mkdir -p redis/rootfs && docker pull redis

# step 2. create the container with a temp name so that we can export it
docker create --name tempredis redis

# step 3. export it into the rootfs directory
docker export tempredis | tar -C redis/rootfs -xf -

# step 4. remove the container now that we have exported
docker rm tempredis

# step 5. prepare runc from https://github.com/opencontainers/runc and cd to redis dir.
cd redis && runc spec

# 现在我们即得到了如下结构的文件
#/redis
#├── config.json
#└── rootfs/
# step 6. 和这里(https://github.com/projectatomic/containerd/blob/master/docs/bundle.md#edits)描述的一样，我们修改config.json里启动命令等参数选项，并将network ns去掉（这类似于net=host的效果），config.json的规格定义参考：https://github.com/opencontainers/runtime-spec/blob/master/config.md

# step 7. 和上面的官方文档有所出入的是，container 1.0启动容器的操作有所变化
# 我们需要用run命令来操作(需要将ctr升级到最新的master分支然后编译).
ctr run -t -d --config /root/redis/config.json --rootfs /root/redis/rootfs redis
```

接下来就是非常眼熟的一些操作了，比如：

```
root@k8s-01:~/redis# ctr containers ls
CONTAINER    IMAGE    RUNTIME
redis        -        io.containerd.runtime.v1.linux
```

时间问题暂时先写到这里，后面再补充。

#### 参考

* https://github.com/containerd/containerd

* https://blog.mobyproject.org/getting-started-with-containerd-a81fa090982f

* https://www.slideshare.net/Docker/containerd-building-a-container-supervisor-by-michael-crosby

* https://container42.com/2017/10/14/containerd-deep-dive-intro/

* https://blog.docker.com/2017/12/containerd-ga-features-2/

* https://blog.mobyproject.org/where-are-containerds-graph-drivers-145fc9b7255

* https://github.com/containerd/containerd/blob/master/docs/ops.md

* https://blog.docker.com/2017/09/kubernetes-containerd-integration/
