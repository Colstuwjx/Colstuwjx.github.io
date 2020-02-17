---
title: "[ 原创 ] Ansible Playbook 执行原理源码解读（一）（粗略）"
date: 2016-05-16T20:00:00+08:00
tags: ["devops", "ansible", "sourcecode"]
categories: ["ansible"]
comments: false
showMeta: false
showAction: false
---

这篇文章主要是分享一下鄙人对 ansible playbook 的执行流程的一些理解。

<!--more-->

#### 前言

之所以想写一篇Ansible源码方面的解读文章，实在是因为今年转岗后专职做开发，然后配置管理工具从Salt转向了Ansible（单个web项目的自部署），对于Ansible这款工具，鄙人在使用方面实在有太多想吐槽的地方（当然，它好使的地方也不少...），尤其是关于这个[issue](https://github.com/ansible/ansible/issues/5952#issuecomment-200780636)简直已经到了反人类的程度，为此，鄙人想通过几篇文章的篇幅，一方面整合一下对于Ansible的整体理解，另外一方面也督促自己实施对该issue的pull request。

在此先列举一下鄙人研读的Ansible代码版本和涉及内容，以供参考：

* [Ansible devel branch](https://github.com/ansible/ansible/commit/95cf095222fef2ea6cde04142021fd7533eabadf) (v2.x)
* 仅包含Playbook部分执行过程源码的解读

话不多说，搞起~

#### 入口

使用过Ansible的人都知道，`ansible-playbook` cli脚本是执行playbook的cmd入口，鄙人也从此处入口开始解读它的执行过程。

```
# ./bin/ansible-playbook

...

# [ Colstuwjx注解 ]  由于执行的脚本是ansible-playbook，
# 因而该执行程序会通过python的import tool导入cli目录的代码
myclass = "%sCLI" % sub.capitalize()
mycli = getattr(__import__("ansible.cli.%s" % sub, fromlist=[myclass]), myclass)

# [ Colstuwjx注解 ] 然后在这里传入命令行参数并执行run方法
cli = mycli(sys.argv)
cli.parse()
exit_code = cli.run()

...

```

接下来，我们转到`cli/playbook.py`，ansible实际将通过这里的代码来执行playbook：

```
# [ Colstuwjx注解 ] 这里注意一下，PlaybookCLI继承了CLI类，CLI类则是cli执行程序通用的一些逻辑的抽象基类
class PlaybookCLI(CLI):
    ''' code behind ansible playbook cli'''
    
    # [ Colstuwjx注解 ] 这里重载了parse方法，加入了一些list tasks、list tags等功能的支持
    def parse(self):

        # create parser for CLI options
        parser = CLI.base_parser(
        ...
        
    # [ Colstuwjx注解 ] 这里是实际执行的代码入口，开头会做一些基本的参数处理以及检查，
    # 尔后执行playbook并整理输出到console
	def run(self):
    	...
        # [ Colstuwjx注解 ] 这里利用variable manager管控ansible的vars作为全局变量
    	# create the variable manager, which will be shared throughout
        # the code, ensuring a consistent view of global variables
        variable_manager = VariableManager()
        variable_manager.extra_vars = load_extra_vars(loader=loader, options=self.options)
        
        # [ Colstuwjx注解 ] 其他代码也很容易看懂，重点在这！
        # create the playbook executor, which manages running the plays via a task queue manager
        pbex = PlaybookExecutor(playbooks=self.args, inventory=inventory,
        variable_manager=variable_manager, loader=loader, options=self.options, passwords=passwords)

        results = pbex.run()
        
        ...
```

在做了一系列的参数检查和实例化，以及inventory的解析等，最终，我们看到了重头戏`PlaybookExecutor`的登场。它是由`cli/playbook.py`里：

`from ansible.executor.playbook_executor import PlaybookExecutor`

这行代码导入。我们不妨转到这块代码继续探索下。

#### PlaybookExecutor类

先上段代码：

```
class PlaybookExecutor:

    '''
    This is the primary class for executing playbooks, and thus the
    basis for bin/ansible-playbook operation.
    '''

	# [ Colstuwjx注解 ] 这里是执行器的初始化方法，很明显，inventory、vars、loader等等，都会传送给它，
    # 然后它来做执行操作，不过值得一提的是，ansible这边写了一个专门的TaskQueueManager来管理task的具体执行，这块鄙人后面讲解
    def __init__(self, playbooks, inventory, variable_manager, loader, options, passwords):
        self._playbooks        = playbooks
        self._inventory        = inventory
        self._variable_manager = variable_manager
        self._loader           = loader
        self._options          = options
        self.passwords         = passwords
        self._unreachable_hosts = dict()

        if options.listhosts or options.listtasks or options.listtags or options.syntax:
            self._tqm = None
        else:
            self._tqm = TaskQueueManager(inventory=inventory, variable_manager=variable_manager, 
            loader=loader, options=options, passwords=self.passwords)

```

在该类实例化后，根据前面一节的执行过程，该去执行run方法了: `pbex.run()`，我们接着看它的run方法是怎么写的：

```
        try:
        	# [ Colstuwjx注解 ] 在这里开始从传入的路径读取文件，并利用Playbook类加载该playbook
            # Playbook类可以帮助将yml文件转换成entries（可以理解为一个个的执行块），并且将需要include的play一块引入，
            # 分别通过Play类和PlaybookInclude类来实例，这里便不再贴对应代码。
            for playbook_path in self._playbooks:
                pb = Playbook.load(playbook_path, variable_manager=self._variable_manager, 
                loader=self._loader)
                
                ...
                
                # [ Colstuwjx注解 ] 这里获取到解析完成后的entries（即play的实例）
                plays = pb.get_plays()
                display.vv(u'%d plays in %s' % (len(plays), to_unicode(playbook_path)))
				
                ...
                
				# [ Colstuwjx注解 ] 这里开始遍历
                for play in plays:
               	...
                
                # [ Colstuwjx注解 ] 这里的post validate查看源码playbook目录下的base.py里Base类的代码post_validate方法，
                # 大致可以猜测为所有参数变量的数据没有办法在最开始时全部校验完，因此需要post validate，
                # 而这里大意就是为了保证后续校验不影响原本的object，所以这里直接拷贝了一份临时play实例来执行
                # Create a temporary copy of the play here, so we can run post_validate
                # on it without the templating changes affecting the original object.
                all_vars = self._variable_manager.get_vars(loader=self._loader, play=play)
                templar = Templar(loader=self._loader, variables=all_vars)
                new_play = play.copy()
                new_play.post_validate(templar)
                
                ...
                
                # [ Colstuwjx注解 ]  _get_serialized_batches 即分批处理逻辑的实现，
                # 默认Ansible会执行所有命中的主机（直接看这个方法的代码就知道了），
                # 也可以指定每个执行批次的具体数量
                # we are actually running plays 
                for batch in self._get_serialized_batches(new_play):
                
                ...
                
                # [ Colstuwjx注解 ]  然后借助TaskQueueManager实际执行它
                # restrict the inventory to the hosts in the serialized batch
            	self._inventory.restrict_to_hosts(batch)
                # and run it...
                result = self._tqm.run(play=play)
                
                # [ Colstuwjx注解 ]  这中间是一些异常和失败主机的处理工作，不再赘述
                ...
                
                # [ Colstuwjx注解 ]  在这里留下retry文件便于下次retry操作
                # send the stats callback for this playbook
                if self._tqm is not None:
                    if C.RETRY_FILES_ENABLED:
                
                ...
                
                # [ Colstuwjx注解 ]  OK, 最后返回结果，单个play执行结束
                return result
```

OK，大体执行工作结束，我们需要回过头来再看下TaskQueueManager是如何具体地执行play（注意，执行单位是play）。TaskQueueManager的代码位于`executor/task_queue_manager.py`文件里，这是一个多进程类，并且提供多个可插拔的loader，由于篇幅有限，鄙人将在下一篇里继续讲解。

Ps. 鄙人之前也写过Saltstack相关的[执行逻辑代码](http://devopstarter.info/yuan-ma-jie-du-saltstackyun-xing-ji-zhi-zhi-job-runtime/)（Minion端），我们对比可以很明显地看到，Ansible的设计更偏向于执行脚本这种，而不是Server-Client这样的服务软件，代码逻辑也相对简单很多，不必有太多的网络和套接字协议等，这或许也是Ansible之所以被更多人采用的原因之一罢。KISS，古人诚不欺我。
