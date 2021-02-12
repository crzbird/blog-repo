+++
title = "spring-cloud-zookeeper服务注册与发现"
author = "crzbird"
github_url = "https://github.com/crzbird"
head_img = ""
created_at = 2021-02-12T21:35:15
updated_at = 2021-02-12T21:35:15
description = "spring-cloud zookeeper 注册中心"
tags = ["java","spring-cloud"]
+++

# Zookeeper服务注册与发现
## Zookeeper简介
ZK同样作为分布式系统中服务注册与发现的一个重要组件，也是稍早期DUBBO的推荐注册中心，与Eureka相比，zookeeper采用CP模式，即在master节点宕机后在集群选举出新master期间，zookeeper集群处于不可用状态。放弃可用性换来强一致性。Zookeeper数据结构可以直观的类比为注册表结构：
/
/node1(value1)  
/node1/node2(value2)
## 在spring-cloud中使用zookeeper作为注册中心
### provider端
POM:
```java
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.apache.zookeeper</groupId>
                    <artifactId>zookeeper</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
            <version>3.4.11</version>
        </dependency>
```
application.yml:
```java
server:
  port: 8004

spring:
  application:
    name: cloud-payment-service
  cloud:
    zookeeper:
      connect-string: your connect host
```
mainClass:
```java
@SpringBootApplication
@EnableDiscoveryClient
public class Payment8004 {

    public static void main(String[] args) {
        SpringApplication.run(Payment8004.class, args);
    }

}
```
controller:
```java
@RestController
@Slf4j
@RequestMapping(value = "/payment")
public class PaymentController {

    @Autowired
    private PaymentService paymentService;

    @Value(value = "${server.port}")
    private String serverPort;

    @GetMapping(value = "/zk")
    public String paymentzk(){
        return "springclloud with zookeeper:"+serverPort+"\t"+ UUID.randomUUID().toString();
    }
```
启动工程：
![image.png](https://image.bytetrick.com/2020/10/image-3c1fbbcf797d4a198ff03f4debf50c6b.png)
### consumer端
POM:
```java
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.apache.zookeeper</groupId>
                    <artifactId>zookeeper</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
            <version>3.4.11</version>
        </dependency>
```

application.yml:
```java
server:
  port: 80
spring:
  application:
    name: cloud-consumerzk-order80
  cloud:
    zookeeper:
      connect-string: your connect host
```
mainClass:
```java
@SpringBootApplication
@EnableDiscoveryClient
public class OrderZKMain80 {

    public static void main(String[] args) {
        SpringApplication.run(OrderZKMain80.class, args);
    }

}
```
config:
```java
@Configuration
public class ApplicationContextConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

}
```

controller:
```java
@RestController
@Slf4j
public class OrderZKController {

    public static final String INVOKE_URL = "http://cloud-payment-service";

    @Autowired
    private RestTemplate restTemplate;

    @GetMapping("/payment/zk")
    public String paymentZk(){
        return restTemplate.getForObject(INVOKE_URL+"/payment/zk",String.class);
    }

}
```
启动工程：
![image.png](https://image.bytetrick.com/2020/10/image-8ae4f886e8084274ab97e10e2bfba3f0.png)
## 总结
仅使用上来说，eureka与zk大同小异，注服务提供者zookeeper注册的均为临时节点，随断联而删除。