---
title: "记一次vps挂马的经历"
date: 2014-09-27T20:00:00+08:00
tags: ["security"]
categories: ["talk"]
comments: false
showMeta: false
showAction: false
---

这是一次奇怪的被黑经历。

<!--more-->

昨天在VPS上帮同事开个账号做SSH代理，在xshell另开一个shell时莫名其妙告诉我密码错误，当时没多想，遂在没退出的终端下另起了一个带不限权的sudo用户操作。

今天没事时候再上VPS看，竟然发现root下多了几个奇怪的程序：

	-rwxr-xr-x   1 root root  1135000 Sep 11 00:41 TSmff*
	-rwxr-xr-x   1 root root  1351181 Aug  7 13:45 TSmww*
    
再查看history发现root用户有键入奇怪的命令：

	313  chmod 0755 /root/TSmww
    314  nohup /root/TSmww > /dev/null 2>&1 &
    315  chmod 0755 /root/TSmff
    316  nohup /root/TSmff > /dev/null 2>&1 &
    317  ls
    318  ps -ef
    319  password
    320  paswd
    321  passwd
    322  ls
    323  ps -ef
    324  ls -al /tmp
    325  ls -al /etc
    326  ls
    327  ps -ef
    328  chmod 0755 /root/TSmww
    329  nohup /root/TSmww > /dev/null 2>&1 &
    330  chmod 0755 /root/TSmff
    331  nohup /root/TSmff > /dev/null 2>&1 &
    332  ps -ef
    333  ls -al /tmp
    334  ls -al /etc
    335  ps -ef
    336  ls -al /etc
    337  ls -al /tmp

嘿！这家伙敲个passwd命令都手抖啊，不过还蛮带劲的啊，用了nohup跑两个奇怪名字的程序，明显是有问题嘛，lsof看看挂了啥端口：

	TSmww  22833 root  5u  IPv4 1299463   0t0  TCP 192.241.239.171:40379->183.56.168.220:6009 (ESTABLISHED)
    
啊哈！6009端口是啥? XWindow! 连上了嘛。
嗯，这哥们的真实IP应该就是这个了，查了下，是广东电信的。
不过仍然存在的疑问是这俩程序干啥事儿了? 不清楚啊。。。

那么，这哥们怎么攻破我的VPS? 看auth.log（centos下是secure日志）大概明白了：

    Sep 26 20:31:46 devops sshd[26021]: error: Could not load host key: /etc/ssh/ssh_host_ed25519_key
    Sep 26 20:31:47 devops sshd[26021]: pam_unix(sshd:auth): authentication failure; logname= uid=0 ..
    Sep 26 20:31:49 devops sshd[26021]: Failed password for root from 111.74.238.99 port 3059 ssh2
    Sep 26 20:31:58 devops sshd[26021]: message repeated 4 times: 
                                        [ Failed password for root from 111.74.238.99 port 3059 ssh2]
    Sep 26 20:31:58 devops sshd[26021]: fatal: Read from socket failed: Connection reset by peer [preauth]
    Sep 26 20:31:58 devops sshd[26021]: PAM 4 more authentication failures; 
                                        logname= uid=0 euid=0 tty=ssh ruser= rhost=111.74.238.99  user=root
    Sep 26 20:31:58 devops sshd[26021]: PAM service(sshd) ignoring max retries; 5 > 3
    Sep 26 20:32:06 devops sshd[26023]: error: Could not load host key: /etc/ssh/ssh_host_ed25519_key
    Sep 26 20:32:13 devops sshd[26025]: error: Could not load host key: /etc/ssh/ssh_host_ed25519_key
    Sep 26 20:32:13 devops sshd[26023]: Accepted password for root from 117.45.158.11 port 2928 ssh2

这个 `ssh_host_ed25519_key` 的host key不断出现，并重复报出错误passwd或者 `reverse mapping checking getaddrinfo for 229.51.174.61.dial.wz.zj.dynamic.163data.com.cn [61.174.51.229] failed - POSSIBLE BREAK-IN ATTEMPT!` 类似的反向映射问题。

可以看出来，这位同学是用的SSH暴力扫描和破解，解决办法当然就是加固SSH策略，禁止ROOT登陆、禁止PASSWD登陆方式，block可疑IP等。

不过，TSmww和TSmff程序除了开远程操控还干了啥呢? 不知道怎么做呢，欸，实力还是很弱啊。
