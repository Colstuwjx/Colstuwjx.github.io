---
title: "源码解读Saltstack运行机制之Job Runtime"
date: 2014-07-30T20:00:00+08:00
tags: ["devops", "salt", "sourcecode"]
categories: ["salt"]
comments: false
showMeta: false
showAction: false
aliases:
    - /yuan-ma-jie-du-saltstackyun-xing-ji-zhi-zhi-job-runtime/
---

鄙人深切以为 “在开源的世界里阅读源码才是深层次理解软件运行机制的不二法门”，本系列即是旨在通过解读（当然，鄙人实力有限，只是粗浅的解读）Saltstack源码的方式，了解Saltstack内部的运行机制，对排障和更好理解和二次开发Saltstack有一定助力。

<!--more-->

本篇文章即解决”salt 如何从启动minion到执行一个job“的困惑，鄙人才疏学浅，有欠缺的地方还请海涵。

> under salt master & minion 0.17.5

### 一个Salt Minion的启动过程

从init.d下的启动脚本开始追溯可以定位启动代码位于salt下的\_\_init\_\_.py初始脚本里的Minion类，它会执行prepare方法来完成一系列的准备工作：

* 读取配置文件，获取一系列的参数变量;
* 调用Minion或MultiMinion方法，在这一步获取grains数据，并尝试和Master认证，Pillar的编译(?) ;
（这里会做一次判断，如果minion配置文件里指向了多个master，则会调用salt.minion.MultiMinion方法来启动多个Minion Server instance，这里暂时只讨论调用Minion方法的情况）
* check_user判断当前调起Minion user是否符合conf文件描述，满足则进行Minion Server的tune in，也就是从初始化完成到真正Online working的微调过程;
* tune_in 是只有Minion Server或者说主Minion进程才能调用的微调方法，其建立Minion Pub\Pull socket，连接到Master的消息总线，并fire event声明该Minion已启动，并开始执行设定的scheduler（如果有），并进入一个无限循环，长期的监听event并去完成对应任务，至此Minion服务算是启动完毕。

```
    def tune_in(self):
        '''
        Lock onto the publisher. This is the main event 
        loop for the minion
        '''
        ……
```

```
        # On first startup execute a state run if configured to do so
        self._state_run()
        time.sleep(.5)

		#wjx add
        #here start the loop!!!
        loop_interval = int(self.opts['loop_interval'])
        while True:

```

### 基于Zeromq的消息总线

Salt Master和Minion的交互主要借助Zeromq完成，所有的Job也是通过Zeromq消息总线来下发和接收。Zeromq通信这块相应的被掩盖在底层接口下，目前鄙人暂时未能搞清具体的运作机理，也就相应带过，后面可能专门写文理解一下。

### 清道夫——Minion Worker
#### \_handle_payload 方法

正如上文所介绍的那样，Salt Minion Server（Main Minion）在每次收到Master下发的Job时均会对应调起一个新的Worker进程（或线程，具体可以在/etc/salt/minion参数里予以设置，默认为 `#multiprocessing: True` ），这个操作便是在_\_handle\_payload 方法里完成。

                
                #wjx add
                #In main loop, Main Minion will call _handle_payload
                #if there is some poll data from Master
                socks = dict(self.poller.poll(
                    loop_interval * 1000)
                )
                if self.socket in socks and socks[self.socket] == zmq.POLLIN:
                    payload = self.serial.loads(self.socket.recv())
                    self._handle_payload(payload)

#### How to handle payload ?

那么，\_handle\_payload方法如何去处理Master端下发的Job呢？答案看源码便明了一二：

    def _handle_payload(self, payload):
        '''
        Takes a payload from the master publisher
        and does whatever the master wants done.
        '''
        {'aes': self._handle_aes,
         'pub': self._handle_pub,
         'clear': self._handle_clear}[payload['enc']](payload['load'],
                                       payload['sig'] if 'sig' in payload else None)
                                                      
实际上是根据传入的payload dict的enc键值来对应调用方法，最常见的应该是\_handle\_aes方法。然后，Minion便根据本地的minion_master.pub来解密数据，得到data字典（这中间有段代码使用到pub key，但是目前没怎么看懂意思）：

        try:
            data = self.crypticle.loads(load)
            
解密之后的数据使用\_handle\_decoded\_payload予以对应处理，在这里，Main Minion调用Multiprocessing模块或者Threading模块来唤醒一个新的Worker予以处理对应的job任务：

		#wjx add
        #There is some difference between Linux and Win
        #Multiprocess or Multi-Thread is def in /etc/salt/minion
        #also, it will be diff if Master called multi-func like :
        #import salt.client
        #local = salt.client.LocalClient()
        #local.cmd( '*', [ 'test.ping','cmd.run' ], [ [],['w'] ] )
        
        instance = self
        if self.opts['multiprocessing']:
            if sys.platform.startswith('win'):
                # let python reconstruct the minion
                # on the other side if we're
                # running on windows
                instance = None
            process = multiprocessing.Process(
                target=target, args=(instance, self.opts, data)
            )
        else:
            process = threading.Thread(
                target=target, args=(instance, self.opts, data)
            )
    
在Main Minion唤起一个Worker去完成对应Job后，实际上这个Job就只跟Worker Minion有关了，Main Minion重新回到监听event->处理Job这样的一个循环。新的Worker由以下入口（或者\_thread\_multi\_return）开始：

	@classmethod
    def _thread_return(cls, minion_instance, opts, data):
        '''
        This method should be used as a threading target, 
        start the actual minion side execution.
        '''
        # this seems awkward at first, but it's a workaround for Windows
        # multiprocessing communication.
        ……
  
这里注意一点，在Minion完成decode过程时便由\_\_load\_modules导入了Minion端的 functions（也可以理解为modules）以及 returners 数据，所以此时便可以直接调用Master下发的job数据中指定的function：

    def __load_modules(self):
        '''
        Return the functions and the returners loaded up from the loader
        module
        '''
        self.opts['grains'] = salt.loader.grains(self.opts)
        functions = salt.loader.minion_mods(self.opts)
        returners = salt.loader.returners(self.opts, functions)
        return functions, returners
        
Minion Worker 调用function执行Job:
        
        #wjx add
        #use success tag to judge the job ret
        ret = {'success': False}
        function_name = data['fun']
        if function_name in minion_instance.functions:
            try:
                func = minion_instance.functions[data['fun']]
                args, kwargs = parse_args_and_kwargs(func, data['arg'], data)
                sys.modules[func.__module__].__context__['retcode'] = 0
                return_data = func(*args, **kwargs)
                
之后便是Minion Worker反馈数据到Master以及Returner的Job归档：

		#wjx add
        #send data to master by _return_pub function
        ret['jid'] = data['jid']
        ret['fun'] = data['fun']
        minion_instance._return_pub(ret)
        if data['ret']:
            ret['id'] = opts['id']
            for returner in set(data['ret'].split(',')):
                try:
                    minion_instance.returners['{0}.returner'.format(
                        returner
                    )](ret)
                except Exception as exc:
                    log.error(
                        'The return failed for job {0} {1}'.format(
                        data['jid'],
                        exc
                        )
                    )

如果一切顺利的话，\_thread\_return方法成功完成，Worker进程或线程便会自然消亡（到达process.join()，终止）。

### 总结

除开细节上可能存在的一些模糊和错误之处需要完善，本文算是告一段落，主要就Minion角度考虑从启动到处理Job的整个运转过程，理解这样一个机制可能对一些Minion Worker的异常僵死的处理有一定助力。

Salt的源码本身是一个优秀的Python编程样例，阅读其源码不单有助于理解Salt的运转机制，而且对Python开发也会有一定帮助，可以的话，鄙人会持续进行源码的挖掘。

下篇文章可能会就salt\salt-run\salt-call等模块亦或是zeromq机制展开讨论，具体内容就看鄙人能不能写出来了……
