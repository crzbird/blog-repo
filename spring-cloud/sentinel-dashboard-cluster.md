+++
title = "sentinel 集群限流"
author = "crzbird"
github_url = "https://github.com/crzbird"
head_img = ""
created_at = 2021-02-12T21:35:15
updated_at = 2021-02-12T21:35:15
description = "Sentinel-dashboard集群流控"
tags = ["java","spring-cloud"]
+++

# Sentinel-dashboard集群流控
## 1.默认情况
Sentinel-dashboard默认情况下，只可对单机进行流控配置。
## 2.启动方式，独立模式&嵌入模式
启动方式
Sentinel 集群限流服务端有两种启动方式：

独立模式（Alone），即作为独立的 token server 进程启动，独立部署，隔离性好，但是需要额外的部署操作。独立模式适合作为 Global Rate Limiter 给集群提供流控服务。
![image.png](https://image.bytetrick.com/2020/10/image-f412b5415b0047f3a5a8ec0e2a1f510c.png)

嵌入模式（Embedded），即作为内置的 token server 与服务在同一进程中启动。在此模式下，集群中各个实例都是对等的，token server 和 client 可以随时进行转变，因此无需单独部署，灵活性比较好。但是隔离性不佳，需要限制 token server 的总 QPS，防止影响应用本身。嵌入模式适合某个应用集群内部的流控。
![image.png](https://image.bytetrick.com/2020/10/image-bcf3cdc0cf364ce48f8c6cc639f125ad.png)

我们提供了 HTTP API 用于在 embedded 模式下转换集群流控身份：

http://<ip>:<port>/setClusterMode?mode=<xxx>
其中 mode 为 0 代表 client，1 代表 server，-1 代表关闭。注意应用端需要引入集群限流客户端或服务端的相应依赖。

在独立模式下，我们可以直接创建对应的 ClusterTokenServer 实例并在 main 函数中通过 start 方法启动 Token Server。
## 项目改造(嵌入模式)
服务实例POM引入：
```java
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-transport-simple-http</artifactId>
            <version>1.6.3</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-cluster-client-default</artifactId>
            <version>1.6.3</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-cluster-server-default</artifactId>
            <version>1.6.3</version>
        </dependency>

```
## Sentinel-dashboard配置：
1.配置集群：
![image.png](https://image.bytetrick.com/2020/10/image-a56f69819f4f4285975de8de43983628.png)
2.配置规则:
![image.png](https://image.bytetrick.com/2020/10/image-fd825b0a01454529b1f8de31458fa0e2.png)
## 至此集群流控生效
