+++
title = "spring-cloud-consul服务注册与发现"
author = "crzbird"
github_url = "https://github.com/crzbird"
head_img = ""
created_at = 2021-02-12T21:35:15
updated_at = 2021-02-12T21:35:15
description = "spring-cloud consul配置中心"
tags = ["java","spring-cloud"]
+++

# Consul简介
Consul是一套开源的分布式服务发现和配置管理系统，由Hashicorp公司推出，采用GoLang开发；提供服务治理，配置中心，控制总线等功能，可根据业务单独使用。采用CP模式，舍弃强一致性，以保证集群的可用性。
# Consul使用
## Provider端：
POM引入：
```java
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-consul-discovery</artifactId>
        </dependency>
```
Application.yml:
```java
spring:
  application:
    name: consul-provider-payment
  cloud:
    consul:
      discovery:
        service-name: ${spring.application.name}
      host: localhost
      port: 8500
```
MainClass:
```java
@SpringBootApplication
@EnableDiscoveryClient
public class Payment8006 {

    public static void main(String[] args) {
        SpringApplication.run(Payment8006.class, args);
    }

}
```
Controller:
```java
@RestController
@RequestMapping(value = "/payment")
@Slf4j
public class PaymentController {


    @Value(value = "${server.port}")
    private String serverPort;

    @GetMapping(value = "/consul")
    public String paymentConsul() {
        return "springcloud with consult:" + this.serverPort;
    }
}
```

## Consumer端：
POM引入：
```java
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-consul-discovery</artifactId>
        </dependency>
```
Application.yml：
```java
spring:
  cloud:
    consul:
      port: 8500
      host: localhost
      discovery:
        service-name: ${spring.application.name}
  application:
    name: cloud-consumer-order
```
MainClass：
```java
@SpringBootApplication
@EnableDiscoveryClient
public class OrderConsulMain80 {

    public static void main(String[] args) {
        SpringApplication.run(OrderConsulMain80.class, args);
    }

}
```
Controller:
```java
@RestController
@RequestMapping(value = "/consumer")
@Slf4j
public class OrderController {

    public static final String INVOKE_URL = "http://consul-provider-payment";

    @Autowired
    private RestTemplate restTemplate;

    @GetMapping(value = "/payment/consul")
    public String paymentInfo() {
        return restTemplate.getForObject(INVOKE_URL + "/payment/consul", String.class);
    }

}
```
## Console控制台
![image.png](https://image.bytetrick.com/2020/11/image-e7eb50116c4c49e48cd0db44968829a2.png)
## 测试
![image.png](https://image.bytetrick.com/2020/11/image-bf64a2fa7b8d4f76a8e180f71f09feb5.png)
