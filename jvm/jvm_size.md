+++
title = "JVM内存设置"
author = "crzbird"
github_url = "https://github.com/crzbird"
head_img = ""
created_at = 2021-02-14T21:35:15
updated_at = 2021-02-14T21:35:15
description = "关于JVM内存设置"
tags = ["java","jvm"]
+++
# JVM内存设置多大合适？
## 根据Java Performance里面的推荐公式来进行设置
|Space  |  Command Line Option  |  Occupancy Factor  |
|:----------|:----------|:----------|
| Java heap    | -Xms and -Xmx    | 3x to 4x old generation space occupancy after full garbage collection    |
| Permanent Generation    | -XX:PermSize -XX:MaxPermSize    | 1.2x to 1.5x permanent generation space occupancy after full garbage collection    |
| Young Generation    | -Xmn    | 1x to 1.5x old generation space occupancy after full garbage collection    |
| Old Generation    |  Implied from overall Java heap size minus the young generation size    | 2x to 3x old generation space occupancy after full garbage collection    |

具体来讲：
Java整个堆大小设置，Xmx 和 Xms设置为老年代存活对象的3-4倍，即FullGC之后的老年代内存占用的3-4倍
永久代 PermSize和MaxPermSize设置为老年代存活对象的1.2-1.5倍。
年轻代Xmn的设置为老年代存活对象的1-1.5倍。
老年代的内存大小设置为老年代存活对象的2-3倍。
### BTW：
1、Sun官方建议年轻代的大小为整个堆的3/8左右， 所以按照上述设置的方式，基本符合Sun的建议。
2、堆大小=年轻代大小+年老代大小， 即xmx=xmn+老年代大小 。 Permsize不影响堆大小。
3、为什么要按照上面的来进行设置呢？ 没有具体的说明，但应该是根据多种调优之后得出的一个结论
### 如何确认老年代存活对象大小？
#### 方式1（推荐/比较稳妥）：
JVM参数中添加GC日志，GC日志中会记录每次FullGC之后各代的内存大小，观察老年代GC之后的空间大小。可观察一段时间内（比如2天）的FullGC之后的内存情况，根据多次的FullGC之后的老年代的空间大小数据来预估FullGC之后老年代的存活对象大小（可根据多次FullGC之后的内存大小取平均值）
#### 方式2：（强制触发FullGC, 会影响线上服务，慎用)
方式1的方式比较可行，但需要更改JVM参数，并分析日志。同时，在使用CMS回收器的时候，有可能不能触发FullGC（只发生CMS GC），所以日志中并没有记录FullGC的日志。在分析的时候就比较难处理。
BTW：使用jstat -gcutil工具来看FullGC的时候， CMS GC是会造成2次的FullGC次数增加。 具体可参见之前写的一篇关于jstat使用的文章
所以，有时候需要强制触发一次FullGC，来观察FullGC之后的老年代存活对象大小。
注：强制触发FullGC，会造成线上服务停顿（STW），要谨慎，建议的操作方式为，在强制FullGC前先把服务节点摘除，FullGC之后再将服务挂回可用节点，对外提供服务
在不同时间段触发FullGC，根据多次FullGC之后的老年代内存情况来预估FullGC之后的老年代存活对象大小.
#### 如何触发FullGC ？
使用jmap工具可触发FullGC
jmap -dump:live,format=b,file=heap.bin <pid> 将当前的存活对象dump到文件，此时会触发FullGC
jmap -histo:live <pid> 打印每个class的实例数目,内存占用,类全名信息.live子参数加上后,只统计活的对象数量. 此时会触发FullGC.

