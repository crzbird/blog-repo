+++
title = "spring-cloud-config配置中心"
author = "crzbird"
github_url = "https://github.com/crzbird"
head_img = ""
created_at = 2021-02-12T21:35:15
updated_at = 2021-02-12T21:35:15
description = "spring-cloud配置中心"
tags = ["java","spring-cloud"]
+++

# Spring Cloud Config简介
Spring Cloud Config provides server-side and client-side support for externalized configuration in a distributed system. With the Config Server, you have a central place to manage external properties for applications across all environments. The concepts on both client and server map identically to the Spring Environment and PropertySource abstractions, so they fit very well with Spring applications but can be used with any application running in any language. As an application moves through the deployment pipeline from dev to test and into production, you can manage the configuration between those environments and be certain that applications have everything they need to run when they migrate. The default implementation of the server storage backend uses git, so it easily supports labelled versions of configuration environments as well as being accessible to a wide range of tooling for managing the content. It is easy to add alternative implementations and plug them in with Spring configuration.
简言之，SpringCloudConfig提供一个跨语言跨环境的配置中心ConfigServer，来统一管理大量微服务的配置。。。（用Nacos不香嘛。。。）
常规高可用部署方式：
![image.png](https://image.bytetrick.com/2020/11/image-e5f4b60f0ea94312b1c0d883707138c2.png)
## 基本使用
### Config Server
将配置托管到git，本文以github作为示例:
![image.png](https://image.bytetrick.com/2020/11/image-245ddbeeca3b44bbaaa9a2a284ea9903.png)
需单独创建一个工程作为配置服务
POM引入：
```java
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
```
Application.yml:
```java
server:
  port: 3344
spring:
  application:
    name: cloud-config-center
  cloud:
    config:
      server:
        git:
          uri: https://github.com/crzbird/spring-cloud-config-repo.git #仓库地址 HTTPS 或 SSH都可
          search-paths:
            - spring-cloud-config-repo #仓库
      label: main #分支
```
MainClass:
```java
@SpringBootApplication
@EnableConfigServer
public class ConfigCenterMain3344 {

    public static void main(String[] args) {
        SpringApplication.run(ConfigCenterMain3344.class, args);
    }

}
```
测试：
![image.png](https://image.bytetrick.com/2020/11/image-1be17728f84241248b1ab70ca3cbfc19.png)
### Config Client
pom引入：
```java
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
```
bootstrap.yml
```java
server:
  port: 3355
spring:
  application:
    name: config-client
  cloud:
    config:
      label: main #分支
      name: config #名
      profile: dev #描述
      uri: http://localhost:3344 #config server 地址
```
MainClass:
```java
@SpringBootApplication
public class ConfigClientMain3355 {
    public static void main(String[] args) {
        SpringApplication.run(ConfigClientMain3355.class);
    }
}
```
Controller:
```java
@RestController
public class ConfigClientController {

    @Value("${config.info}")
    private String configInfo;

    @GetMapping(value = "/configInfo")
    public String getConfigInfo() {
        return this.configInfo;
    }

}
```
测试client获取config：
![image.png](https://image.bytetrick.com/2020/11/image-3de4709dbf8446abb74ce36df22ce6f7.png)
## 配置刷新
### Config Client
POM引入：
```java
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
```
bootstrap.yml:
暴露actuator端点
```java
management:
  endpoints:
    web:
      exposure:
        include: "*"
```
需要刷新的配置所在的类加入@RefreshScope注解
```java
@RestController
@RefreshScope
public class ConfigClientController {

    @Value("${config.info}")
    private String configInfo;

    @GetMapping(value = "/configInfo")
    public String getConfigInfo() {
        return this.configInfo;
    }

}
```
POST http://localhost:3355/actuator/refresh 刷新配置：
![image.png](https://image.bytetrick.com/2020/11/image-5d1db121c19340c8865961d4c968335d.png)
再次访问：
![image.png](https://image.bytetrick.com/2020/11/image-8e7d4a7486c34b51a0974a573d262b42.png)
配置完成刷新。
