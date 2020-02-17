---
title: "kube-dns实现原理的解读"
date: 2018-01-20T20:00:00+08:00
tags: ["kubernetes", "cloud-native", "devops", "sourcecode"]
categories: ["cloud-native"]
comments: false
showMeta: false
showAction: false
---

最近一直在做类似skydns的dns服务发现相关的研究和代码实现，也就有了兴趣想解读下kube-dns的实现过程，学习和参考下社区的做法。

<!--more-->

#### 源码结构

如何从头开始阅读一份开源项目的源码呢？一般来说，我们会从执行入口顺腾摸瓜地依次八下去，这里强烈推荐难易大佬的[k8s源码阅读笔记](http://dockone.io/article/895)，条理非常清晰，解读的方式也很值得学习，我们这里不妨也参考难易的方法进行kube-dns代码的阅读。

我们首先需要从github仓库clone下来[kube-dns](http://)的源代码。不难看到其代码入口cmd目录下除开测试组件，主要是dnsmasq、dns、sidecar这三块组件的代码：

![kube-dns-dir](/images/2018/Jan/dir.jpg)

OK，由此我们产生了第一个也是最核心的问题：

> Q1: kube-dns、dnsmasq-nanny、sidecar分别实现了什么功能?

我们不妨先从kube-dns这块开始，接着往下读。

#### kube-dns

我们可以看到kube-dns的入口代码[dns.go](https://github.com/kubernetes/dns/blob/master/cmd/kube-dns/dns.go#L35)，这里面的main函数具体内容如下：

```
...

func main() {
	config := options.NewKubeDNSConfig()
	config.AddFlags(pflag.CommandLine)

	flag.InitFlags()
    ...

	server := app.NewKubeDNSServerDefault(config)
	server.Run()
}
```

很显然，初始化kube-dns的配置然后根据命令行参数重载，并随后通过`NewKubeDNSServerDefault`方法拿到kube-dns server的实例，并运行它的`Run`方法。在这部分的阅读过程中，我们也产生了如下的疑问：

> Q2: `NewKubeDNSConfig`方法里有哪些具体可配置的参数？

> Q3: `NewKubeDNSServerDefault`方法做了哪些事情？

戳进去`NewKubeDNSConfig`方法的[具体代码位置](https://github.com/kubernetes/dns/blob/master/cmd/kube-dns/app/options/options.go#L56)：

```
func NewKubeDNSConfig() *KubeDNSConfig {
	return &KubeDNSConfig{
    	// NOTE(Colstuwjx): 集群域名的后缀，所有这类域名解析均由kube-dns管理和解析
		ClusterDomain:      "cluster.local.",
		HealthzPort:        8081,
		DNSBindAddress:     "0.0.0.0",
		DNSPort:            53,
        
        // ?
		InitialSyncTimeout: 60 * time.Second,

		Federations: make(map[string]string),

		ConfigMapNs: api.NamespaceSystem,
		ConfigMap:   "", // default to using command line flags

		ConfigPeriod: 10 * time.Second,
		ConfigDir:    "",

		NameServers: "",
	}
}
```

鄙人不禁又产生了新的疑问：

> Q4: 为什么还有ConfigMap相关的配置以及同步参数？

> Q5: NameServers的配置会影响什么？

OK，没关系，先放着这些问题，接着来看下`NewKubeDNSServerDefault`方法都做了啥：

```
func NewKubeDNSServerDefault(config *options.KubeDNSConfig) *KubeDNSServer {
    ...

	var configSync dnsconfig.Sync
	switch {
	case config.ConfigMap != "" && config.ConfigDir != "":
    ...
    }
    
	return &KubeDNSServer{
		domain:         config.ClusterDomain,
		healthzPort:    config.HealthzPort,
		dnsBindAddress: config.DNSBindAddress,
		dnsPort:        config.DNSPort,
		nameServers:    config.NameServers,
		kd:             dns.NewKubeDNS(kubeClient, config.ClusterDomain, config.InitialSyncTimeout, configSync),
	}
}
```

啊哈，前面的问题似乎有些眉目了。

> 为什么还有ConfigMap相关的配置以及同步参数？

> A4: 原来如此，前面ConfigMap的配置是用来做kube-dns本身配置的动态同步

那这具体执行的`NewKubeDNS`又是如何实现的呢？看看[代码](https://github.com/kubernetes/dns/blob/master/pkg/dns/dns.go#L123)：

```
	kd := &KubeDNS{
		kubeClient:          client,
		domain:              clusterDomain,
		cache:               treecache.NewTreeCache(),
		cacheLock:           sync.RWMutex{},
		nodesStore:          kcache.NewStore(kcache.MetaNamespaceKeyFunc),
		reverseRecordMap:    make(map[string]*skymsg.Service),
		clusterIPServiceMap: make(map[string]*v1.Service),
		domainPath:          util.ReverseArray(strings.Split(strings.TrimRight(clusterDomain, "."), ".")),
		initialSyncTimeout:  timeout,

		configLock: sync.RWMutex{},
		configSync: configSync,
	}

	kd.setEndpointsStore()
	kd.setServicesStore()
```

咦，这里又不禁冒出来几个问题：

> Q6: 这里的cache是做什么的？

> Q7: 这里的`setEndpointsStore`、`setServicesStore`做了啥？

我们继续看下各自的实现: [TreeCache](https://github.com/kubernetes/dns/blob/master/pkg/dns/treecache/treecache.go#L26)、[setEndpointsStore](https://github.com/kubernetes/dns/blob/master/pkg/dns/dns.go#L227)、[setServicesStore](https://github.com/kubernetes/dns/blob/master/pkg/dns/dns.go#L209)，不难得出Q6和Q7的答案：

> 这里的cache是做什么的？

> A6: 这里的cache通过一个公共的TreeCache结构维护，应该是充当本地cache的角色

> 这里的`setEndpointsStore`、`setServicesStore`做了啥？

> A7: 这里的两个setXXXStore设置了对所有namespace的`services`及`endpoints`的list watch句柄，订阅相应的变化，并实现了对应的CURD callback：`handleEndpointAdd`、`handleEndpointUpdate`、`handleEndpointDelete`，Q6的本地cache也即是为了这些数据的本地缓存服务（包括上锁等）

OK，回过头来，读完了`NewKubeDNSServerDefault`大致的调用过程，我们也就知道了：

> `NewKubeDNSConfig`方法里有哪些具体可配置的参数？

> A2: 包括了集群域名、健康检查配置、DNS监听地址及端口、ConfigMap动态同步的参数、配置文件目录及upstream nameserver配置等

> `NewKubeDNSServerDefault`方法做了哪些事情？

> A3: `NewKubeDNSServerDefault`初始化了dns配置的同步方式，通过执行`NewKubeDNS`实例化`KubeDNS`对象，包括本地cache的初始化等，最后返回一个`KubeDNSServer`的实例对象

那么，[`Run`方法](https://github.com/kubernetes/dns/blob/master/cmd/kube-dns/app/server.go#L117)具体又执行了哪些逻辑呢？

```
func (server *KubeDNSServer) Run() {
	...
	setupSignalHandlers()
    
	server.startSkyDNSServer()
	server.kd.Start()
	server.setupHandlers()

	glog.V(0).Infof("Status HTTP port %v", server.healthzPort)
	if server.nameServers != "" {
		glog.V(0).Infof("Upstream nameservers: %s", server.nameServers)
	}
	glog.Fatal(http.ListenAndServe(fmt.Sprintf(":%d", server.healthzPort), nil))
}
```

从上面代码不难看出`Run`方法实现了如下逻辑：

* `setupSignalHandlers`注册了收到信号量时的处理句柄；
* `startSkyDNSServer`则应该是通过现有配置启动skydns服务，实现DNS解析的功能；
* `kd.Start`则是执行了kube-dns自身控制器的启动，继续跟踪到[kube-dns pkg](https://github.com/kubernetes/dns/blob/master/pkg/dns/dns.go#L145)里的实现部分，不难理解，这里即是启动了对`services`和`endpoints`变更监听及处理的controller，它们将会把变更的部分同步到本地cache，此外，还将执行configMap的[同步routine](https://github.com/kubernetes/dns/blob/master/pkg/dns/dns.go#L152)，实现上面提到的kube-dns配置的动态同步和加载；
* `setupHandlers`即启动了`/readiness`和`/cache`的endpoint，并监听在healthcheck配置的端口；

我们不妨再深入看看`startSkyDNSServer`具体做了哪些事情：

```
func (d *KubeDNSServer) startSkyDNSServer() {
	...
	skydnsConfig := &server.Config{
		Domain:  d.domain,
		DnsAddr: fmt.Sprintf("%s:%d", d.dnsBindAddress, d.dnsPort),
	}
	if d.nameServers != "" {
		for _, nameServer := range strings.Split(d.nameServers, ",") {
			server, err := validateNameServer(nameServer)
			if err != nil {
				glog.Fatalf("nameserver '%s' is invalid: %v\n", nameServer, err)
			}
			skydnsConfig.Nameservers = append(skydnsConfig.Nameservers, server)
		}
	}
	server.SetDefaults(skydnsConfig)
	s := server.New(d.kd, skydnsConfig)
	if err := metrics.Metrics(); err != nil {
		glog.Fatalf("Skydns metrics error: %s", err)
	} else if metrics.Port != "" {
		glog.V(0).Infof("Skydns metrics enabled (%v:%v)", metrics.Path, metrics.Port)
	} else {
		glog.V(0).Infof("Skydns metrics not enabled")
	}

	go s.Run()
}
```

还是蛮清晰的，这里根据kube-dns的配置生成`skydnsConfig`，并随后根据之前配置的nameserver配置skydns server启动时的upstream nameserver，这也就解答了Q5提出的问题。

> A5：`nameservers`配置了的话，在kube-dns启动时会设置upstream的nameserver列表，这里upstream nameserver的含义即当kube-dns(背后做事的即SkyDNS)发现非clusterDomain的dns解析时会递归到upstream nameserver

此外，`server.New(d.kd, skydnsConfig)`实例化skydns  server的时候传入了`d.kd`作为其backend，我们深挖下去不难发现，skydns里[这块](https://github.com/kubernetes/dns/blob/master/vendor/github.com/skynetservices/skydns/server/backend.go#L9)是定义了一个统一的backend interface，抽象了需要实现的方法`Records`和`ReverseRecord`，回过头来看kubedns这块的代码，确实是实现了这两个接口：

```
func (kd *KubeDNS) Records(name string, exact bool) (retval []skymsg.Service, err error) {
	...

	path := util.ReverseArray(segments)
	records, err := kd.getRecordsForPath(path, exact)

	if err != nil {
		return nil, err
	}

    ...
}

...
```

而且这里面调用的方法`getRecordsForPath`，其方法体里执行的`kd.cache.GetEntry`这行正是消费之前定义的本地cache，也即是读取实时维护的本地cache来得到dns记录解析的数据依据：

```
func (kd *KubeDNS) getRecordsForPath(path []string, exact bool) ([]skymsg.Service, error) {
	...

	...
		kd.cacheLock.RLock()
		defer kd.cacheLock.RUnlock()
		if record, ok := kd.cache.GetEntry(key, path[:len(path)-1]...); ok {
			glog.V(3).Infof("Exact match %v for %v received from cache", record, path[:len(path)-1])
			return []skymsg.Service{*(record.(*skymsg.Service))}, nil
		}

		...
	}

	...
}
```

回到上面执行的skydns启动部分，还有一块是实例化metrics对象，进而调用的是pkg/dnsmasq/metric.go里面的代码，如下这部分即获取metric的核心部分：

```
func (mc *metricsClient) getSingleMetric(name string) (int64, error) {
	msg := new(dns.Msg)
	msg.Id = dns.Id()
	msg.RecursionDesired = false
	msg.Question = make([]dns.Question, 1)
	msg.Question[0] = dns.Question{
		Name:   name,
		Qtype:  dns.TypeTXT,
		Qclass: dns.ClassCHAOS,
	}

	in, _, err := mc.dnsClient.Exchange(msg, mc.addrPort)
	if err != nil {
		return 0, err
	}

	if len(in.Answer) != 1 {
		return 0, fmt.Errorf("invalid number of Answer records for %s: %d",
			name, len(in.Answer))
	}

	if t, ok := in.Answer[0].(*dns.TXT); ok {
		glog.V(4).Infof("Got valid TXT response %+v for %s", t, name)
		if len(t.Txt) != 1 {
			return 0, fmt.Errorf("invalid number of TXT records for %s: %d",
				name, len(t.Txt))
		}

		value, err := strconv.ParseInt(t.Txt[0], 10, 64)
		if err != nil {
			return 0, err
		}

		return value, nil
	}

	return 0, fmt.Errorf("missing TXT record for %s", name)
}

```

看上去是通过dnsClient请求dnsmasq server的形式采集到一些dns请求的metric，这部分我们留到dnsmasq和sidecar组件的解读部分再来细说。

至此，我们初略读完了kube-dns部分的启动过程，并基本解答了除开Q1以外的所有疑问，总结一下的话kube-dns基本上是这样的启动流程：

* 通过执行`options.NewKubeDNSConfig`方法和命令行传参初始化Kube-DNS配置；
* 然后通过执行`app.NewKubeDNSServerDefault(config)`初始化配置文件的配置方式（配置文件形式或者从ConfigMap动态配置），并通过`kd.setEndpointsStore()`、`kd.setServicesStore()`方法初始化`endpoints`和`services`的本地cache（本地cache是通过构造TreeCache这样的结构体来实现维护的）；
* 最终，通过执行`server.Run()`方法，初始化信号量的处理句柄，启动skydns服务器并以kube-dns的实现作为backend（数据消费自之前构建和持续维护的本地cache），里面还启动了一个metric endpoint，暴露出dnsmasq上DNS请求相关的监控指标。随后，启动endpoints和services的变更监听及callback处理的controller从而实时维护本地cache数据的准确性，并通过执行`kd.startConfigMapSync`方法，在配置了ConfigMap的情况下实现kube-dns配置的动态加载

#### dnsmasq、sidecar

kube-dns整体的逻辑理顺了，剩下的dnsmasq和sidecar部分实现了怎样的逻辑自然也很好解读。

`nanny`，顾名思义，即是保姆的意思，因此cmd目录下的[dnsmasq部分](https://github.com/kubernetes/dns/blob/master/cmd/dnsmasq-nanny/main.go#L80)很显然即是包装了dnsmasq运行时的组件，管理和维护dnsmasq作为kube-dns前置的dns缓存。

我们不妨跟上文解读kube-dns类似，来看看代码入口做了哪些事情：

```
func main() {
	parseFlags()
	glog.V(0).Infof("opts: %v", opts)

	sync := config.NewFileSync(opts.configDir, opts.syncInterval)

	dnsmasq.RunNanny(sync, opts.RunNannyOpts)
}
```

代码挺简单的，很显然，`config.NewFileSync`即是通过读取ConfigMap的数据，动态同步并生成dnsmasq对应的配置文件，并根据配置参数确认是否需要重启生效。而如下所示的`dnsmasq.RunNanny`方法：

```
// RunNanny runs the nanny and handles configuration updates.
func RunNanny(sync config.Sync, opts RunNannyOpts) {
	defer glog.Flush()

	currentConfig, err := sync.Once()
	if err != nil {
		glog.Errorf("Error getting initial config, using default: %v", err)
		currentConfig = config.NewDefaultConfig()
	}

	nanny := &Nanny{Exec: opts.DnsmasqExec}
	nanny.Configure(opts.DnsmasqArgs, currentConfig)
	if err := nanny.Start(); err != nil {
		glog.Fatalf("Could not start dnsmasq with initial configuration: %v", err)
	}

	configChan := sync.Periodic()

	for {
		select {
		case status := <-nanny.ExitChannel:
			glog.Flush()
			glog.Fatalf("dnsmasq exited: %v", status)
			break
		case currentConfig = <-configChan:
			if opts.RestartOnChange {
				glog.V(0).Infof("Restarting dnsmasq with new configuration")
				nanny.Kill()
				nanny = &Nanny{Exec: opts.DnsmasqExec}
				nanny.Configure(opts.DnsmasqArgs, currentConfig)
				nanny.Start()
			} else {
				glog.V(2).Infof("Not restarting dnsmasq (--restartDnsmasq=false)")
			}
			break
		}
	}
}
```

即是同步dnsmasq的配置并通过`nanny.Start()`启动dnsmasq服务。`nanny.Start()`方法内容则比较清晰，便是通过`exec`启动dnsmasq，这里需要指出的是，dnsmasq启动时上游nameserver的配置，正是根据ConfigMap里是否配置了[config.StubDomains](https://github.com/kubernetes/dns/blob/master/pkg/dnsmasq/nanny.go#L70)以及[config.UpstreamNameservers](https://github.com/kubernetes/dns/blob/master/pkg/dnsmasq/nanny.go#L79)来生成相应的配置。

[K8S官方文档](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/)里讲述的指定一组特定的`StubDomains`由xxx nameserver解析以及配置上游nameserver的功能均由dnsmasq这块的配置实际落地生效！

至此，dnsmasq便成功启动了，它作为kube-dns的前置部分，担任dns缓存和其他非cluster domain的解析请求的转发工作。

那么，我们再来看看sidecar做了什么事情。

```
func main() {
	options := sidecar.NewOptions()
    configureFlags(options, pflag.CommandLine)

	...

	server := sidecar.NewServer()
	server.Run(options)
}
```

同样是读取配置参数，并启动了一个sidecar的server。而查看`server.Run(options)`的具体实现：

```
// Run the server (does not return)
func (s *server) Run(options *Options) {
	s.options = options
	glog.Infof("Starting server (options %+v)", *s.options)

	for _, probeOption := range options.Probes {
		probe := &dnsProbe{DNSProbeOption: probeOption}
		s.probes = append(s.probes, probe)
		probe.Start(options)
	}

	s.runMetrics(options)
}
```

这里根据配置的[options](https://github.com/kubernetes/dns/blob/master/pkg/sidecar/options.go#L36)指定需要对哪些endpoint执行探测，options的配置也很简单，通过传入的命令行参数指定，执行`configureFlags`实例化得到：

```
...
	flagSet.Var(
		(*probeOptions)(&opt.Probes), "probe",
		"probe the given DNS server with the DNS name and export probe"+
			" metrics and healthcheck URI. Specified as"+
			" <label>,<server>,<dns name>[,<interval_seconds>][,<type>]."+
			" Healthcheck url will be exported under /healthcheck/<label>."+
			" interval_seconds is optional."+
			" This option may be specified multiple times to check multiple servers."+
			" <type> is one of ANY, A, AAAA, SRV."+
			" Example: 'mydns,127.0.0.1:53,example.com,10,A'.")
```

最后，即通过`runMetrics`方法里的[InitializeMetrics](https://github.com/kubernetes/dns/blob/master/pkg/sidecar/metrics.go#L91)方法，实例化并启动一个prometheus的metrics端点，提供相应dns记录解析的监控指标。

至此，我们也终于能回答第一个问题了：

> kube-dns、dnsmasq-nanny、sidecar分别实现了什么功能?

> A1：

>（1）kube-dns包装了skydns server的启动，并通过监听services和endpoints的变更事件且同步到本地cache的形式，实现了一个实时级别的k8s集群内service和pod等端点的DNS服务发现；

>（2）dnsmasq-nanny则是通过包装dnsmasq，实现了kube-dns前置的dns缓存，并通过监听ConfigMap动态生成配置，将配置了的StubDomains转发给指定的nameservers，将其他域名的解析递归到配置好的Upstream nameservers；

>（3）sidecar部分代码则是实现了可配置的DNS探测，并采集对应的监控指标暴露出来供prometheus消费

上述的解析过程和[kubedns文档](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/)里描述的也正是不谋而合：

![kube-dns](/images/2018/Jan/dns.png)

完。
