+++
title = "sentinel dashboard 持久化到nacos"
author = "crzbird"
github_url = "https://github.com/crzbird"
head_img = ""
created_at = 2021-02-12T21:35:15
updated_at = 2021-02-12T21:35:15
description = "sentinel dashboard 持久化到nacos"
tags = ["java","spring-cloud"]
+++

# Sentinel简介（使用方式不在本文内）
在目前微服务流行的项目架构下，以流量为切入点，提供流控，熔断降级，负载保护等纬度以保证系统整体稳定。（其实如果找到这篇文章，必然对sentinel有了一定了解，并且被CV-CODER坑过）。
## Sentinel-dashbord
Sentinel-dashboard提供流控配置、监控的可视化web界面，可根据系统负载状况及场景即时配置服务流控策略，默认情况下规则是保存在内存中，服务下线后重置。
## Nacos持久化
需要对源码进行轻度改造，以下是具体步骤。
### 1.拉取sentinel工程
https://github.com/alibaba/Sentinel
### 2.找到dashboard工程
![image.png](https://image.bytetrick.com/2020/10/image-64e4c60671ef447cad14df1ed6c78db5.png)
### 3.修改POM
![image.png](https://image.bytetrick.com/2020/10/image-8e2227e352634fc484c1480e154fe0e9.png)
将scope从test改为provided
### 4.将test下nacos CV到main
![image.png](https://image.bytetrick.com/2020/10/image-fe0fab1138ad4076a979e49e48d80337.png)
### 5.修改FlowControllerV2
![image.png](https://image.bytetrick.com/2020/10/image-bf91e71fafb34aa7bb4ca04ae0ba9864.png)
将default发布者和提供者替换为nacos。
### 6.改造sidebar.html
将V1改为V2：
![image.png](https://image.bytetrick.com/2020/10/image-7bc469b8c8854cc1a61b3bd7e90fff4e.png)
### 7.clean package，至此dashboard-nacos持久化已完成。
## 后话：轻度解析。
### 1.NacosConfig
![image.png](https://image.bytetrick.com/2020/10/image-2dd02767fcc7495bb0522ea07aa3ca17.png)
其中写死nacos地址:localhost:8848，需要根据实际修改。
### 2.NacosConfigUtil
![image.png](https://image.bytetrick.com/2020/10/image-82ac825e85c247b087adf4e9753b7462.png)
定义nacos中流控规则data常量。
### 3.FlowRuleNacosPublisher
![image.png](https://image.bytetrick.com/2020/10/image-a35dc2b849ae4240bb9fd91b44290349.png)
顾名思义，流控规则发布者，推送dashboard配置的策略持久化到nacos。
具体方法在FlowControllerV2中：![image.png](https://image.bytetrick.com/2020/10/image-1a2d043873c74635bc4d27d7c49c15ba.png)
流控规则的增删改均会调用此方法推送规则。
### 4.flowRuleNacosProvider
![image.png](https://image.bytetrick.com/2020/10/image-35a2f83f30e4439aaad9b58c173605ec.png)
流控规则提供者，从nacos拉取规则到dashboard。
具体调用在FlowControllerV2中：
![image.png](https://image.bytetrick.com/2020/10/image-3fb9dbd4121747cc96a3d7574d9dff9a.png)
先通过应用标识从nacos拉取规则(同时需保证Nacos高可用)，并存入内存![image.png](https://image.bytetrick.com/2020/10/image-7dc2a63f3b544195b23cb87e9d10482c.png)
同时返回流控规则到dashboard。
### 总结
持久化方式可能会随版本变动而改变，需关注[Sentinel官方WIKI](https://github.com/alibaba/Sentinel/wiki)。宏观上流程并不复杂，接下来会对sentinel实现原理进行一定程度上源码分析。