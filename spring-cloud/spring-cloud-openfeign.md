+++
title = "spring-cloud-openfeign http rpc 调用"
author = "crzbird"
github_url = "https://github.com/crzbird"
head_img = ""
created_at = 2021-02-12T21:35:15
updated_at = 2021-02-12T21:35:15
description = "spring-cloud openfeign rpc"
tags = ["java","spring-cloud"]
+++

# OpenFeign简介
Feign is a declarative web service client. It makes writing web service clients easier. To use Feign create an interface and annotate it. It has pluggable annotation support including Feign annotations and JAX-RS annotations. Feign also supports pluggable encoders and decoders. Spring Cloud adds support for Spring MVC annotations and for using the same HttpMessageConverters used by default in Spring Web. Spring Cloud integrates Ribbon and Eureka, as well as Spring Cloud LoadBalancer to provide a load-balanced http client when using Feign.
Feign 是一个声明式webService客户端，可以使调用更加简便，采用大量注解驱动，并且支持插件（比如编码和解码等）；像之前使用RestTemplate进行服务接口调用
![image.png](https://image.bytetrick.com/2020/11/image-0c6c10bd1fca4869adbe91b441abe136.png)
极不优雅，并在调用多个服务情况下变得难以维护。
## 官方示例
```java
@SpringBootApplication
@EnableFeignClients
public class WebApplication {

	public static void main(String[] args) {
		SpringApplication.run(WebApplication.class, args);
	}

	@FeignClient("name")
	static interface NameService {
		@RequestMapping("/")
		public String getName();
	}
}
```
## 使用方式
POM引入：
```java
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
```
Application.yml:
无必须配置项，可根据业务自行选取：
![image.png](https://image.bytetrick.com/2020/11/image-205868cffa424d5b897588346cac0a78.png)

MainClass：
```java
@SpringBootApplication
@EnableFeignClients
public class OrderFeignMain80 {

    public static void main(String[] args) {
        SpringApplication.run(OrderFeignMain80.class, args);
    }

}
```
配置调用过程日志：
```java
@Configuration
public class FeignConfig {

    @Bean
    Logger.Level feignLoggerLevel(){
        return Logger.Level.FULL;
    }

}
```
![image.png](https://image.bytetrick.com/2020/11/image-83e653a37c4f44a1bf5ed976dba41e6b.png)
定义service接口以及fallbackFactory：
```java
@FeignClient(value = "CLOUD-PAYMENT-SERVICE", fallbackFactory = PaymentFallbackFactory.class)
public interface PaymentFeignService {

    @GetMapping(value = "/payment/get/{id}")
    public CommonResult getPaymentById(@PathVariable(value = "id") Long id);

    @GetMapping(value = "/payment/feign/timeout")
    public String paymentFeignTimeout();

    @GetMapping(value = "/payment/lb")
    public String getPaymentLB();

}
```
```java
@Component
@Slf4j
public class PaymentFallbackFactory implements FallbackFactory<PaymentFeignService> {


    @Override
    public PaymentFeignService create(Throwable throwable) {
        return new PaymentFeignService() {
            @Override
            public CommonResult getPaymentById(Long id) {
                log.error("err:",throwable);
                return new  CommonResult(500,"调用失败：fallback");
            }

            @Override
            public String paymentFeignTimeout() {
                return null;
            }

            @Override
            public String getPaymentLB() {
                return null;
            }
        };
    }
}
```
调用方使用：

```java
@RestController
@RequestMapping(value = "/consumer")
public class OrderController {

    @Autowired
    private PaymentFeignService paymentFeignService;

    @GetMapping(value = "/payment/get/{id}")
    public CommonResult paymentGetById(@PathVariable(value = "id") Long id) {
        return paymentFeignService.getPaymentById(id);
    }

    @GetMapping(value = "/payment/feign/timeout")
    public String paymentFeignTimeout() {
        return paymentFeignService.paymentFeignTimeout();
    }

    @GetMapping(value = "/payment/lb")
    public String paymentLb() {
        return paymentFeignService.getPaymentLB();
    }

}
```
调用测试：
![image.png](https://image.bytetrick.com/2020/11/image-e559642a0ac04e008febd45d3790fe50.png)
使用的负载均衡器同样是Ribbon
![image.png](https://image.bytetrick.com/2020/11/image-3ab3871f2f7d4fc0a2d30ae6899e0973.png)
## 总结
使用openFeign进行服务调用，相对使用原来restTemplate来说更加方便，更加面向对象（面向接口），提供fallback接口对调用异常兜底。