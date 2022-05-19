---
title: 'Jstat 使用'
date: 2020-12-29 14:24:19
tags: JVM
cover: https://sailliao.oss-cn-beijing.aliyuncs.com/img/wallhaven-m9e2lm.jpg
---

# 类加载统计
## jstat -class PID
```bash
jstat -class 2428959
Loaded  Bytes    Unloaded  Bytes     Time   
9987    18477.8  66        97.0      29.86
```
* Loaded 加载class的数量
* Bytes 所占用空间大小
* Unloaded 未加载数量
* Bytes 未加载占用空间
* Time 时间

# 编译统计
## jstat -compiler PID
```bash
jstat -compiler 2428959
Compiled  Failed  Invalid   Time    FailedType  FailedMethod
8275      0       0         42.43   0          
```
* Compiled：编译数量
* Failed：失败数量
* Invalid：不可用数量
* Time：时间
* FailedType：失败类型
* FailedMethod：失败的方法

# 垃圾回收统计
## jstat -gc PID
```bash
jstat -gc  2428959
S0C    S1C     S0U    S1U      EC        EU        OC         OU        MC       MU       CCSC    CCSU     YGC     YGCT    FGC    FGCT     GCT   
2112.0 2112.0  0.0    1710.9   16960.0   9395.5    42300.0    35860.5   57472.0  54766.3  7040.0  6528.7   1381    4.761   4      0.348    5.110
```

* S0C：第一个幸存区的大小
* S1C：第二个幸存区的大小
* S0U：第一个幸存区的使用大小
* S1U：第二个幸存区的使用大小
* EC：伊甸园区的大小
* EU：伊甸园区的使用大小
* OC：老年代大小
* OU：老年代使用大小
* MC：方法区大小
* MU：方法区使用大小
* CCSC:压缩类空间大小
* CCSU:压缩类空间使用大小
* YGC：年轻代垃圾回收次数
* YGCT：年轻代垃圾回收消耗时间
* FGC：老年代垃圾回收次数
* FGCT：老年代垃圾回收消耗时间
* GCT：垃圾回收消耗总时间

# 堆内存统计
## jstat -gccapacity PID
```bash
jstat -gccapacity 2428959
NGCMN    NGCMX     NGC      S0C     S1C       EC         OGCMN      OGCMX       OGC         OC           MCMN     MCMX       MC           CCSMN    CCSMX       CCSC     YGC      FGC 
10240.0  156288.0  21184.0  2112.0  2112.0    16960.0    20480.0    312704.0    42300.0     42300.0      0.0      1099776.0  57472.0      0.0      1048576.0   7040.0   1381     4
```
* NGCMN：新生代最小容量
* NGCMX：新生代最大容量
* NGC：当前新生代容量
* S0C：第一个幸存区大小
* S1C：第二个幸存区的大小
* EC：伊甸园区的大小
* OGCMN：老年代最小容量
* OGCMX：老年代最大容量
* OGC：当前老年代大小
* OC:当前老年代大小
* MCMN:最小元数据容量
* MCMX：最大元数据容量
* MC：当前元数据空间大小
* CCSMN：最小压缩类空间大小
* CCSMX：最大压缩类空间大小
* CCSC：当前压缩类空间大小
* YGC：年轻代gc次数
* FGC：老年代GC次数

# 新生代垃圾回收统计
## jstat -gcnew PID
```bash
jstat -gcnew 2428959
S0C     S1C      S0U    S1U     TT MTT  DSS      EC        EU       YGC     YGCT  
2112.0  2112.0   0.0    1710.9  1  15   1056.0   16960.0   9728.0   1381    4.761
```
* S0C：第一个幸存区大小
* S1C：第二个幸存区的大小
* S0U：第一个幸存区的使用大小
* S1U：第二个幸存区的使用大小
* TT:对象在新生代存活的次数
* MTT:对象在新生代存活的最大次数
* DSS:期望的幸存区大小
* EC：伊甸园区的大小
* EU：伊甸园区的使用大小
* YGC：年轻代垃圾回收次数
* YGCT：年轻代垃圾回收消耗时间

# 新生代内存统计
## jstat -gcnewcapacity PID
```bash
jstat -gcnewcapacity 2428959
NGCMN      NGCMX       NGC      S0CMX     S0C     S1CMX     S1C       ECMX        EC       YGC      FGC 
10240.0    156288.0    21184.0  15616.0   2112.0  15616.0   2112.0   125056.0     16960.0  1381     4
```
* NGCMN：新生代最小容量
* NGCMX：新生代最大容量
* NGC：当前新生代容量
* S0CMX：最大幸存1区大小
* S0C：当前幸存1区大小
* S1CMX：最大幸存2区大小
* S1C：当前幸存2区大小
* ECMX：最大伊甸园区大小
* EC：当前伊甸园区大小
* YGC：年轻代垃圾回收次数
* FGC：老年代回收次数

# 老年代垃圾回收统计
## jstat -gcold PID
```bash
jstat -gcold 2428959
MC       MU        CCSC     CCSU       OC          OU        YGC      FGC    FGCT     GCT   
57472.0  54766.3   7040.0   6528.7     42300.0     35860.5   1381     4      0.348    5.110
```
* MC：方法区大小
* MU：方法区使用大小
* CCSC:压缩类空间大小
* CCSU:压缩类空间使用大小
* OC：老年代大小
* OU：老年代使用大小
* YGC：年轻代垃圾回收次数
* FGC：老年代垃圾回收次数
* FGCT：老年代垃圾回收消耗时间
* GCT：垃圾回收消耗总时间

# 老年代内存统计
## jstat -gcoldcapacity PID
```bash
jstat -gcoldcapacity 2428959
OGCMN       OGCMX        OGC         OC       YGC      FGC    FGCT     GCT   
20480.0     312704.0     42300.0     42300.0  1381     4      0.348    5.110
```
* OGCMN：老年代最小容量
* OGCMX：老年代最大容量
* OGC：当前老年代大小
* OC：老年代大小
* YGC：年轻代垃圾回收次数
* FGC：老年代垃圾回收次数
* FGCT：老年代垃圾回收消耗时间
* GCT：垃圾回收消耗总时间

# 元数据空间统计
## jstat -gcmetacapacity PID
```bash
jstat -gcmetacapacity 2428959
MCMN       MCMX         MC             CCSMN      CCSMX         CCSC     YGC      FGC    FGCT     GCT   
0.0        1099776.0    57472.0        0.0        1048576.0     7040.0   1381     4      0.348    5.110
```
* MCMN: 最小元数据容量
* MCMX：最大元数据容量
* MC：当前元数据空间大小
* CCSMN：最小压缩类空间大小
* CCSMX：最大压缩类空间大小
* CCSC：当前压缩类空间大小
* YGC：年轻代垃圾回收次数
* FGC：老年代垃圾回收次数
* FGCT：老年代垃圾回收消耗时间
* GCT：垃圾回收消耗总时间

# 总结垃圾回收统计
## jstat -gcutil PID
```bash
S0     S1     E      O      M      CCS     YGC     YGCT      FGC    FGCT     GCT   
0.00   81.01  59.67  84.78  95.29  92.74   1381    4.761     4      0.348    5.110
```
* S0：幸存1区当前使用比例
* S1：幸存2区当前使用比例
* E：伊甸园区使用比例
* O：老年代使用比例
* M：元数据区使用比例
* CCS：压缩使用比例
* YGC：年轻代垃圾回收次数
* FGC：老年代垃圾回收次数
* FGCT：老年代垃圾回收消耗时间
* GCT：垃圾回收消耗总时间

# JVM编译方法统计
##  jstat -printcompilation PID
```bash
jstat -printcompilation 2428959
Compiled  Size   Type   Method
8275      213    1      sun/util/locale/LocaleUtils caseIgnoreMatch
```
* Compiled：最近编译方法的数量
* Size：最近编译方法的字节码数量
* Type：最近编译方法的编译类型
* Method：方法名标识

# 文章来源
[jstat命令查看jvm的GC情况 （以Linux为例）](http://blog.itpub.net/31543790/viewspace-2657093/)