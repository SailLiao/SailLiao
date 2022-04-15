---
title: VisualVM 的使用
date: 2021-09-08 15:00:02
tags: 虚拟机
cover: https://img9.51tietu.net/pic/2019-091402/avh2tfl0aknavh2tfl0akn.jpg
---

# 下载和插件安装

从 [https://visualvm.github.io/download.html]下载后，点击 visualvm.exe 可能会说找不到 jdk，需要配置下,在 etc 目录，visualvm.conf

```
visualvm_jdkhome="D:/Program/X64/Java"
```
必须是 / 不然不能生效

从 [https://visualvm.github.io/pluginscenters.html]选取对应的版本的链接。

![](1.png)

然后在 Tools -> Plugins -> Settings 里面，右侧 Edit 编辑 URL。然后在 Available Plugins 里面勾选 Visual GC 和 BTree Workbench,点击安装，安装可能会出错，因为是访问GitHub，可以设置代理。

安装后，重新打开进程可以看见 Visual GC

![](2.png)

## BTrace 动态日志跟踪

它的作用是在不停止目标程序运行的前提下， 通过HotSpot虚拟机的HotSwap技术动态加入原本并不存在的调试代码。 
这项功能对实际生产中的程序很有意义： 
1. 经常遇到程序出现问题， 但排查错误的一些必要信息， 譬如方法参数、 返回值等， 在开发时并没有打印到日志之中， 以至于不
得不停掉服务， 通过调试增量来加入日志代码以解决问题。 
2. 当遇到生产环境服务无法随便停止时， 缺一两句日志导致排错进行不下去是一件非常郁闷的事情。

在进程ID上右键可以选择 dump 当时的堆内存的情况。

### HotSwap

代码热替换技术， HotSpot虚拟机允许在不停止运行的情况下， 更新已经加载的类的代码。

![](3.png)

# 连接远程主机

Remote -> Add Remote Host...

![](4.png)

远程主机需要配置额外的参数
```
JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=9004 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Djava.net.preferIPv4Stack=true -Djava.rmi.server.hostname=39.108.70.86"
```
关键的自己的外网IP 39.108.70.86 和端口 9004

# 总结

其实很少用，知道有这个东西就行，需要时拿出来看看，本地一般都测不出来优化问题，远程连接可能会受限于安全组之类的。