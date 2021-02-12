+++
title = "spring-cloud-hystrix断路器"
author = "crzbird"
github_url = "https://github.com/crzbird"
head_img = ""
created_at = 2021-02-12T21:35:15
updated_at = 2021-02-12T21:35:15
description = "spring-cloud hystrix断路器"
tags = ["java","spring-cloud"]
+++
# Hystrix简介
Hystrix是一个延迟和容错库，旨在隔离对远程系统、服务和第三方库的访问点，停止级联故障，并使复杂的分布式系统在不可避免的故障中具有弹性。也就是说，hystrix能够有效的解决在分布式系统环境下由于某服务故障引起服务雪崩的问题。
如图：
![image.png](https://image.bytetrick.com/2020/11/image-44bd2a34058d4cf29cadee3858c41ffa.png)
在微服务架构中，较大情况出现扇出的服务调用链路。如果G服务由于某些原因超时或不可用，则对于B C D应用的负载会积压进而级联影响服务A，最终造成整个系统不可用，这就是所谓的“服务雪崩”；为解决此类情况，则需要引入熔断、降级组件。
## 基本概念
服务降级：当服务超时，运行异常、被熔断、线程池满等会触发降级fallback。
服务熔断：服务运行异常达到一定阈值（可配置）后，在一定窗口期内服务将会被熔断，熔断期间服务不会被调用，直接fallback。
服务限流：限制服务在单位时间内被请求的次数。
## 使用方式
### Provider端
POM引入：
```java
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
```
Application.yml：
无必须配置项，可根据业务按需配置
![image.png](https://image.bytetrick.com/2020/11/image-1e153e25e2b3401d9edc0f6d25ab52e1.png)
MainClass：
```java
@SpringBootApplication
@EnableEurekaClient
@EnableCircuitBreaker
public class  PaymentHystrixMain8005 {

    public static void main(String[] args) {
        SpringApplication.run(PaymentHystrixMain8005.class, args);
    }
}
```
案例方法：
```java
@Service
@Slf4j
public class PaymentService {
    @HystrixCommand(fallbackMethod = "paymentCircuitBreaker_fallback", commandProperties = {
            @HystrixProperty(name = "circuitBreaker.enabled", value = "true"),//是否开启断路器
            @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "10"),//请求次数
            @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "10000"),//时间窗口期
            @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "60"),//失败率达到多少后断路
    })//根据此配置：在10秒内请求10次，异常率达到60，则开启熔断10秒，在下一个10窗口期放行一次请求，如果成功则关闭断路器，失败则继续断路
    public String paymentCircuitBreaker(Integer id) {
        log.info(Thread.currentThread().getName()+ "paymentCircuitBreaker\t"+id);
        if (id < 0) {
            throw new RuntimeException("*****id 不能负数");
        }
        String serialNumber = IdUtil.simpleUUID();
        return Thread.currentThread().getName() + "\t 调用成功，流水号：" + serialNumber;

    }

    public String paymentCircuitBreaker_fallback(Integer id) {
        return "id 不能负数 fallback";
    }
}
```
fallbackMethod指定兜底方法，当方法执行异常将使用此方法兜底；@HystrixProperty指定规则，如上方法：在每10此请求中如果异常率达到60%则熔断10000ms，在窗口期后会断路器会处于“半开（half open）”状态，放行一次请求，如果此次请求成功则关闭断路器，失败则继续熔断指定窗口期。
@HystrixProperty配置项可在HystrixPropertiesManager中查找
![image.png](https://image.bytetrick.com/2020/11/image-f827b6648d8e4005ba1a0c81c33bc4bb.png)
![image.png](https://image.bytetrick.com/2020/11/image-ae84529975084975b96d9466f32bb325.png)
### Consumer端
POM引入：
```java
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
```
Application.yml：
```java
feign:
  hystrix:
    enabled: true
  client:
    config:
      default:
        connectTimeout: 3000 # feign 的超时设置
        readTimeout: 3000
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 3000 # 设置hystrix的超时时间为3000ms, 之后才调用降级方法,需要同时设置feign和hystrix的超时时间，否则会按最小的时间超时
```
MainClass：
```java
@SpringBootApplication
@EnableFeignClients
@EnableHystrix
public class OrderHystrixMain80 {

    public static void main(String[] args) {
        SpringApplication.run(OrderHystrixMain80.class, args);
    }

}
```
示例方法：
```java
@RestController
@RequestMapping(value = "/consumer")
@DefaultProperties(defaultFallback = "payment_Global_FallbackMethod")
public class OrderHystrixController {

    @Autowired
    private PaymentHystrixService paymentHystrixService;

    @GetMapping(value = "/payment/hystrix/ok/{id}")
    public String paymentInfo_OK(@PathVariable(value = "id") Integer id) {
        return paymentHystrixService.paymentInfo_OK(id);
    }


    @GetMapping(value = "/payment/hystrix/timeout/{id}")
    @HystrixCommand(fallbackMethod = "paymentTimeOutFallbackMethod", commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "1500")
    })//HystrixCommand针对调用方的方法fallback ，在feign接口上的fallbackfactory针对调用服务提供方的方法fallback
    public String paymentInfo_Timeout(@PathVariable(value = "id") Integer id) {
        return paymentHystrixService.paymentInfo_Timeout(id);

    }

    public String paymentTimeOutFallbackMethod(Integer id) {
        return "consumer 80 fallback";
    }

    public String payment_Global_FallbackMethod() {
        return "consumer 80 global fallback";
    }
}
```
注意：在@FeignClient接口中配置的fallback是对于调用服务过程中出现的异常进行兜底；@HystrixCommand的fallbackMethod是对本方法异常的兜底。
## Hystrix-dashboar
Hystrix-dashboar提供近乎实时的调用监控，以图表形式展示请求的执行信息统计。
### 使用方式
POM引入：
```java
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```
Application.yml:
```java
server:
  port: 9001
```
MainClass:
```java
@SpringBootApplication
@EnableHystrixDashboard
public class HystrixDashboardMain9001 {

    public static void main(String[] args) {
        SpringApplication.run(HystrixDashboardMain9001.class, args);
    }
}
```
注意：
被监控的服务需要引入spring-boot-starter-actuator，如果是新版本的hystrix需要配置监控路径：
```java
    @Bean
    public ServletRegistrationBean getServlet() {
        HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();
        ServletRegistrationBean registrationBean = new ServletRegistrationBean(streamServlet);
        registrationBean.setLoadOnStartup(1);
        registrationBean.addUrlMappings("/hystrix.stream");
        registrationBean.setName("HystrixMetricsStreamServlet");
        return registrationBean;
    }
```
启动Hystrix-dashboar工程并填入监控地址：
![image.png](https://image.bytetrick.com/2020/11/image-c15d38db70d94e5b93ab35e935a62d2b.png)
![image.png](https://image.bytetrick.com/2020/11/image-e8e0cdd0a0984cee988cdbee859fd319.png)

监控说明：
![image.png](https://image.bytetrick.com/2020/11/image-582396d2d5a941d2b6f7c3a6088f30ea.png)
