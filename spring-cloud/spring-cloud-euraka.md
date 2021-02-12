+++
title = "spring-cloud-eureka服务注册与发现"
author = "crzbird"
github_url = "https://github.com/crzbird"
head_img = ""
created_at = 2021-02-12T21:35:15
updated_at = 2021-02-12T21:35:15
description = "spring-cloud eureka注册中心"
tags = ["java","spring-cloud"]
+++
# Eureka服务注册发现
## Eureka简介
官网描述：Eureka is a REST (Representational State Transfer) based service that is primarily used in the AWS cloud for locating services for the purpose of load balancing and failover of middle-tier servers. We call this service, the Eureka Server. Eureka also comes with a Java-based client component,the Eureka Client, which makes interactions with the service much easier. The client also has a built-in load balancer that does basic round-robin load balancing. At Netflix, a much more sophisticated load balancer wraps Eureka to provide weighted load balancing based on several factors like traffic, resource usage, error conditions etc to provide superior resiliency.
总的来说，eureka主要提供服务治理、服务注册与发现。包含两大组件1.eureka-server（作为服务端），2.eureka-client（作为客户端）。客户端每30秒（默认）向服务端发送心跳包，如果在60秒（默认）内server未接收client的心跳，则将其剔除。eureka提供自我保护机制：如果eureka在短时间内丢失大量服务心跳，则会触发自我保护，并不会剔除服务。因为网络分区的不稳定性，短时间内丢失大量服务心跳，可能是由于此server实例与client网络延迟或连接丢失，但是client可能健康的、可用的，由此Eureka是属于CAP中的AP模式，保证可用性和分区容错性放弃强一致性。
## Eureka使用
### server端
eureka server需自行建立工程。
引入POM：
```java
        <!--eureka server-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>

        <!--boot web actuator-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```
application.yml:
```java
server:
  port: 7001
spring:
  application:
    name: eureka-server7001
eureka:
  client:
    fetch-registry: false #server 不需要拉取服务
    register-with-eureka: false #不需要注册为服务
    service-url:
      defaultZone: http://eureka7002.com:7002/eureka,http://eureka7003.com:7003/eureka
  instance:
    hostname: eureka7001.com
```
MainClass:
```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaMain7001 {

    public static void main(String[] args) {
        SpringApplication.run(EurekaMain7001.class, args);
    }

}
```
以上示例为单机伪集群，三实例相互注册，如下：
![image.png](https://image.bytetrick.com/2020/10/image-89e50e7943a44f8993702c7edabc6d54.png)
### client-provider端
POM引入：
```java
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```
application.yml:
```java
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka,http://eureka7002.com:7002/eureka,http://eureka7003.com:7003/eureka
  instance:
    instance-id: payment8001
    prefer-ip-address: true
    #最后一次心跳90秒内没发心跳则被剔除
    lease-expiration-duration-in-seconds: 90
    #30秒一次心跳包
    lease-renewal-interval-in-seconds: 30
```

MainClass:
```java
@SpringBootApplication
@EnableEurekaClient
public class Payment8001 {

    public static void main(String[] args) {
        ConfigurableApplicationContext run = SpringApplication.run(Payment8001.class, args);
    }
}
```
Controller:
```java
@RestController
@Slf4j
@RequestMapping(value = "/payment")
public class PaymentController {

    @Autowired
    private PaymentService paymentService;

    @Value(value = "${server.port}")
    private String serverPort;

    @Autowired
    private DiscoveryClient discoveryClient;

    @GetMapping(value = "/lb")
    public String getPaymentLB() {
        return serverPort;
    }

}
```
注册到eureka后：
![image.png](https://image.bytetrick.com/2020/10/image-697a45ad8ec542959bec7f13ba7be45e.png)
### client-consumer端：
POM引入：
```java
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```
application.yml:
```java
server:
  port: 80
eureka:
  client:
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka,http://eureka7002.com:7002/eureka,http://eureka7003.com:7003/eureka
  instance:
    instance-id: consumer80
```
MainClass:

```java
@SpringBootApplication
@EnableEurekaClient
public class OrderMain80 {

    public static void main(String[] args) {
        SpringApplication.run(OrderMain80.class, args);
    }
}
```
config:
@Configuration
public class ApplicationContextConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

}
controller:
```java
@RestController
@RequestMapping(value = "/consumer")
@Slf4j
public class OrderController {

    public static final String PAYMENT_URL = "http://CLOUD-PAYMENT-SERVICE/";

    @Autowired
    private RestTemplate restTemplate;

    @GetMapping(value = "/payment/lb")
    public CommonResult<String> paymentLB() {
        String lb = restTemplate.getForObject(PAYMENT_URL + "/payment/lb", String.class);
        return new CommonResult<>(200, "suc", lb);
    }
}
```
启动后：
![image.png](https://image.bytetrick.com/2020/10/image-91bc8c7701b64c8c876bb5e087ac959e.png)
## 测试
访问接口：localhost/consumer/payment/lb

![image.png](https://image.bytetrick.com/2020/10/image-e6efb99cc7dd4240b76d90c948a65e0b.png)
![image.png](https://image.bytetrick.com/2020/10/image-a0a868fdd1b3472fbee1f94a038a5317.png)
![image.png](https://image.bytetrick.com/2020/10/image-93fe704a7a8a48349bade0af72c2a60e.png)
## 总结
以上eureka最基本搭建和使用，下一篇 openFeign.
