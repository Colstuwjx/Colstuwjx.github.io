---
title: "Python 开发之print结果写入文件"
date: 2014-05-27T20:00:00+08:00
tags: ["devops", "python"]
categories: ["python"]
comments: false
showMeta: false
showAction: false
---

编写程序时，调试是一个很重要的步骤，为此，各种开发语言也建立了自身完善的Debug方法。最常见和普适的方法便是通过Echo或Print将需要了解的变量或返回值输出到Console，以判断当前程序的执行情况。

<!--more-->

Python同样提供print方法（Python2作为关键字使用，Python3则是作为方法调用，即print xx 和 print(xx)）。值得一提的是，Python提供的print方法支持str、int、object和list等数据类型。当无法在Console获得程序输出时，可以通过直接将所需信息输出到文件的方式来进行调试：

    list = ['a','b']
    f = open(filepath, 'w')
    print >>f,list
    f.close()
    
添加上述代码即可完成print结果写入到文件。
