+++
title = "Eureka、Zookeeper、Consul关于CAP对比"
author = "crzbird"
github_url = "https://github.com/crzbird"
head_img = ""
created_at = 2021-02-12T21:35:15
updated_at = 2021-02-12T21:35:15
description = "Eureka、Zookeeper、Consul关于CAP对比"
tags = ["java","spring-cloud","register center"]
+++

# 何为CAP
CAP最初由Eric Brewer大神提出，在分布式系统中，存在三大要素：Consistency(一致性)、Availability(可用性)、Partition tolerance(分区容忍性)，而在CAP中只能较好的满足两个。
## CAP拆分
CA:满足强一致性与可用性，也就是单点集群，舍弃分区容忍性从而扩展性不强。
CP:满足强一致性与分区容忍性，也就是在节点间数据同步期间集群处于不可用状态；例如Zookeeper集群：当master节点宕机后，在集群选举期间，整个Zookeeper集群是不可用的，直到新的master选举完成，才对外提供服务。
AP:满足可用行与分区容忍性，集群允许短暂的数据不一致，而保证可用性；例如Eureka集群，当某个节点断联、宕机后，集群还是能够向外提供服务，尽管各个节点中数据可能不一致。
## 经典CAP圈圈图
![image.png](https://image.bytetrick.com/2020/11/image-a919cf46c02f4def92302e29d40257f7.png)
## Eureka（AP）
![image.png](https://image.bytetrick.com/2020/11/image-afce6808b26949bfa9d50e1a7466ae35.png)
## Zookeeper和Consul（CP）
![image.png](https://image.bytetrick.com/2020/11/image-efc9b3f55b5c45d2af2dad530d9cb680.png)
## 横向对比
|组件名|语言|CAP|健康检查|对外接口暴露方式|SpringCloud集成|
|-------|-------|-------|-------|-------|-------|
|Eureka|Java|AP|支持（可配置）|HTTP|集成|
|Zookeeper|Java|CP|支持|客户端|集成|
|Consul|GO|CP|支持|HTTP/DNS|集成|
