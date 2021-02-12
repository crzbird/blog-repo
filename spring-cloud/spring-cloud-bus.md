+++
title = "spring-cloud-bus消息总线"
author = "crzbird"
github_url = "https://github.com/crzbird"
head_img = ""
created_at = 2021-02-12T21:35:15
updated_at = 2021-02-12T21:35:15
description = "spring-cloud消息总线"
tags = ["java","spring-cloud"]
+++
# Spring Cloud Bus简介
Spring Cloud Bus links the nodes of a distributed system with a lightweight message broker. This broker can then be used to broadcast state changes (such as configuration changes) or other management instructions. A key idea is that the bus is like a distributed actuator for a Spring Boot application that is scaled out. However, it can also be used as a communication channel between apps. This project provides starters for either an AMQP broker or Kafka as the transport.
spring cloud bus 将分布式节点与一个轻量级的消息服务连接，以广播状态的改变，目前只支持AMQP或者kafka，本文将以RabbitMQ配合上文的配置中心作为示例。
## 基本使用
### Config Server
POM引入：
```java
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-bus-amqp</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
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
          uri: https://github.com/crzbird/spring-cloud-config-repo.git
          search-paths:
            - spring-cloud-config-repo
      label: main
  rabbitmq:
    host: your host
    port: 5672
    username: guest
    password: guest
management:
  endpoints:
    web:
      exposure:
        include: "bus-refresh"
```
### Config Client
POM引入：
```java
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-bus-amqp</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```
bootstrap.yml:
```java
server:
  port: 3355
spring:
  application:
    name: config-client
  cloud:
    config:
      label: main
      name: config
      profile: dev
      uri: http://localhost:3344
  rabbitmq:
    host: your host
    port: 5672
    username: guest
    password: guest
```
## 测试
![image.png](https://image.bytetrick.com/2020/11/image-a54ed22295c14535a258fb8468905264.png)
POST localhost:3344/actuator/bus-refresh 触发BUS刷新。
重新访问Config Client查看配置结果：
![image.png](https://image.bytetrick.com/2020/11/image-2ee296e41cb74bb695f84cf9cd4ef64a.png)
Config Client控制台：
![image.png](https://image.bytetrick.com/2020/11/image-c6bd9a7e3bac443da6a676327fe600ac.png)
## 刷新指定服务
POST http://localhost:3344/actuator/bus-refresh/config-client:3355 spring.application.name:port来刷新指定服务
## 总结
Spring Cloud Bus 在RabbitMQ创建一个springCloudBus TOPIC交换机，所有Server Client各自创建一个队列并以RoutingKey：#与此交换机绑定
![image.png](https://image.bytetrick.com/2020/11/image-9353d03800944a37bc9c95541938164f.png)
，当Config Server触发bus-refresh时，往交换机发送刷新通知并广播给所有Config Client触发配置刷新。