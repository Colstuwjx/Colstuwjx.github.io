---
title: "结合Salt-output实现Job前端下发的友好展现"
date: 2014-08-29T20:00:00+08:00
tags: ["devops", "salt"]
categories: ["salt"]
comments: false
showMeta: false
showAction: false
---

最近一段时间一直在做类似salt-halite的Salt作业发布Web工具，其中很重要的一点就是如何友好的展现输出，一直也是很想直接调用Salt原生的salt-output系统，没能做成。在通过Github上给官方提Issue，以及群里运维大神苦咖啡的热心帮助下，终有所得，不敢有所藏，愿分享之。

<!--more-->

##### 怎么用?

用个例子来展示如何使用salt-output：

```
import salt.client
import salt.output
import salt.config
    
__opts__ = salt.config.client_config('/etc/salt/master')

#run salt job
l = salt.client.LocalClient()
ret = l.cmd('Minion_01', 'state.highstate', [])

#get salt-output
fmt_ret = salt.output.display_output(ret, 'text', __opts__)
```


但是如果直接调用salt-output,无法拿到相应的str返回，`fmt_ret`值便是None，这样的话意义不大（默认salt-output输出对象是终端），可以通过简单修改代码实现逻辑：

	#salt/output/__init__.py
    ...
    output_filename = opts.get('output_file', None)
    try:
        if output_filename is not None:
            with salt.utils.fopen(output_filename, 'a') as ofh:
                ofh.write(display_data)
                ofh.write('\n')
            	return
        if display_data:
            print(display_data)
            
            #add one line
            return display_data
    except IOError as exc:
        # Only raise if it's NOT a broken pipe
        if exc.errno != errno.EPIPE:
            raise exc

这样的话`display_output`函数便可以返回一个job执行结果的字符串数据，不过由于返回值只是一个bash终端下可识别的带颜色的数据串，所以仍需要进一步加工以友好的反馈到前端：


	import re
    matcher = re.compile(r'\x1b.*?m')
    final_output = matcher.sub('', fmt_output)
    

这样便拿到了最终需要的job返回字符串数据了!!紧接着，便可以使用pre标签渲染，展现出一个友好的Job执行返回结果的页面！
