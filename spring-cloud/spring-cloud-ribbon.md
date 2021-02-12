+++
title = "spring-cloud-ribbon负载均衡器"
author = "crzbird"
github_url = "https://github.com/crzbird"
head_img = ""
created_at = 2021-02-12T21:35:15
updated_at = 2021-02-12T21:35:15
description = "spring-cloud ribbon lb"
tags = ["java","spring-cloud"]
+++

# Ribbon简介
Spring Cloud Ribbon是Netflix ribbon实现的一套**客户端**负载均衡工具，也就是作用在调用方的负载均衡。
# 内置LB算法
1.BestAvailableRule：跳过被熔断的服务，并选取最低请求负载的服务。
2.PredicateBasedRule：在经过AbstractServerPredicate实例（需自行实现）过滤后的服务中轮询选取服务（Round robin）。
3.RoundRobinRule：轮询选取。
4.RandomRule：随机算去。
5.RetryRule：重试。
6.WeightedResponseTimeRule:根据响应时间动态生成权重并轮询选取。
# 配置
Configuration：
选取LB算法
```java
@Configuration
public class MySelfRule {

    @Bean
    public IRule iRule(){
        return new RandomRule();
    }
}
```
配置restTemplate：
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

MainClass：
```java
@SpringBootApplication
@EnableEurekaClient
@RibbonClient(name = "CLOUD-PAYMENT-SERVICE",configuration = MySelfRule.class)
public class OrderMain80 {

    public static void main(String[] args) {
        SpringApplication.run(OrderMain80.class, args);
    }
}
```
# 大体配置过程
RibbonClientConfigurationRegistrar类实现ImportBeanDefinitionRegistrar注册Bean定义：
将主类上@RibbonClient定义注册
![image.png](https://image.bytetrick.com/2020/11/image-a4a83910e10542458bb5412daace7a48.png)
进入RibbonAutoConfiguration，创建LoadBalancerClient
![image.png](https://image.bytetrick.com/2020/11/image-189278690433426f840d39d7d04abd45.png)
AsyncLoadBalancerAutoConfiguration装配异步负载均衡拦截器
![image.png](https://image.bytetrick.com/2020/11/image-df447547bb094119bc28122437d972cc.png)
# RestTemplate调用过程
实际调用发生在InterceptingClientHttpRequest的executeInternal方法
![image.png](https://image.bytetrick.com/2020/11/image-cb7546c3e85a40fdbbcb091129d5114d.png)
创建执行器并执行
![image.png](https://image.bytetrick.com/2020/11/image-760e4e46a65f475d97a13429dadaabd9.png)
使用之前定义的负载均衡器执行请求request
![image.png](https://image.bytetrick.com/2020/11/image-0449f9c07142478284acd51b7937ffe1.png)

进入之前自定义的负载均衡器中，获取服务
![image.png](https://image.bytetrick.com/2020/11/image-58436e2044654bd4a0ef03e3d2d2443b.png)
根据serviceId获取负载均衡器并根据负载均衡策略获取本次调用的服务后执行请求
![image.png](https://image.bytetrick.com/2020/11/image-a884ee3fdff94b8ba2e6315399f8ea65.png)