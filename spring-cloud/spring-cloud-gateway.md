+++
title = "spring-cloud-gateway网关"
author = "crzbird"
github_url = "https://github.com/crzbird"
head_img = ""
created_at = 2021-02-12T21:35:15
updated_at = 2021-02-12T21:35:15
description = "spring-cloud网关"
tags = ["java","spring-cloud"]
+++

# Spring Cloud Gateway简介
Gateway提供了一个在Spring WebFlux上构建的API网关。Spring Cloud Gateway旨在提供一种简单而有效的方法来路由到api，并为它们提供跨领域的关注，例如:安全性、监视/度量和弹性。
注意！！！：
![image.png](https://image.bytetrick.com/2020/11/image-bd7fe2ce252041cfb55e6401ce73e857.png)
Gateway是基于springboot2.x，webFlux和响应类库的，许多同步类库并不支持；并且gateway底层容器为netty，传统servlet以及打成war包后将会不生效。
官方提供运行流程图：
![image.png](https://image.bytetrick.com/2020/11/image-dd6640bf20c841aaade19054d2decfda.png)
## 基本使用
POM引入：
```java
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>
        <!--如果需要监控-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```
Application.yml(使用nacos作为注册中心和配置中心):
```java
server:
  port: 9527

spring:
  application:
    name: cloud-gateway
  cloud:
    nacos:
      config:
        server-addr: localhost:8848
        group: DEFAULT_GROUP
        file-extension: yml
    gateway:
      loadbalancer:
        ribbon:
          enable: false #关闭ribbon负载均衡器（ribbon LB为同步）,切换为 ReactiveLoadBalancerClientFilter
      routes:
        - id: payment_route # 路由的id,没有规定规则但要求唯一,建议配合服务名
          #匹配后提供服务的路由地址
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/get/** # 断言，路径相匹配的进行路由
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/lb/** #断言,路径相匹配的进行路由
          #- After=2020-04-29T17:17:27.888+08:00[Asia/Shanghai]
          #- Cookie=username,ddd
          #- Header=X-Request-Id, \d+
          #- Query=username, \d+
      discovery:
        locator:
          enabled: true
eureka:
  instance:
    hostname: cloud-gateway-service
  client:
    fetch-registry: true
    register-with-eureka: true
    service-url:
      defaultZone: http://localhost:7001/eureka,http://eureka7002.com:7002/eureka,http://eureka7003.com:7003/eureka
management:
  endpoint:
    gateway:
      enabled: true
  endpoints:
    web:
      exposure:
        include: gateway
```
配置说明：
1.- id: payment_route # 路由的id,没有规定规则但要求唯一,建议配合服务名。
2.uri: lb://CLOUD-PAYMENT-SERVICE uri中使用“lb”，将会是用spring cloud LoadBalancerClient去解析后面跟着的服务名，如uri: lb://CLOUD-PAYMENT-SERVICE，负载均衡器将会从注册中心获取CLOUD-PAYMENT-SERVICE。
3.predicates:
- Path=/payment/get/** # 断言，路径相匹配的进行路由。
MainClass:
```java
@SpringBootApplication
@EnableEurekaClient
public class GateWayMain9527 {

    public static void main(String[] args) {
        SpringApplication.run(GateWayMain9527.class, args);
    }
}
```
Configuration:
```java
@Configuration
public class GateWayConfig {
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```
启动测试：
![image.png](https://image.bytetrick.com/2020/11/image-0ad42607c7da4c0a8c25dca5c590f8dc.png)
![image.png](https://image.bytetrick.com/2020/11/image-3a664a1e42a843e09dd7b9da0bca0ec0.png)
## 断言工厂介绍
上文仅仅根据断言predicates:- Path=/payment/lb/** 负载均衡到后端服务。事实上Gateway提供较为完整的路由断言工厂：
![image.png](https://image.bytetrick.com/2020/11/image-044acc85f7754a0882a14b2f731a4c04.png)
```java
            #- Path=/payment/lb/** #断言,路径相匹配的进行路由
            #- After=2020-01-02T17:17:27.888+08:00[Asia/Shanghai] #请求时间在此时间之后
            #- Before=2020-01-30T17:17:27.888+08:00[Asia/Shanghai] #请求时间在此时间之前
            #- Between=2020-01-02T17:17:27.888+08:00[Asia/Shanghai], 2020-01-30T17:17:27.888+08:00[Asia/Shanghai] #请求时间在此区间
            #- Cookie=key,value #带有cookie：key=value
            #- Header=X-Request-Id, \d+ #带有Header:X-Request-Id: \d+
            #- Query=key, \d+ #请求带有query：https://bytetrick.com/path?key=\d+
            #- Host={pattern}.somehost.com #请求为**.somehost.com,{pattern}可在ServerWebExchange使用
            #- Method=GET,POST #请求method
            #- RemoteAddr=192.168.1.1/24 #ip断言
```
Weight Route Predicate Factory:
以下示例 /weight将会 80%路由到百度，20%路由到谷歌
```java
        - id: wight_high # 路由的id,没有规定规则但要求唯一,建议配合服务名
          uri: http://baidu.com #匹配后提供服务的路由地址
          predicates:
            - Path=/weight # 断言，路径相匹配的进行路由
            - Weight=group1, 8 #权重：配置所属group1，权重8
        - id: wight_low # 路由的id,没有规定规则但要求唯一,建议配合服务名
          uri: http://google.com #匹配后提供服务的路由地址
          predicates:
            - Path=/weight # 断言，路径相匹配的进行路由
            - Weight=group1, 2 #权重：配置所属group1，权重2
```
## 自定义GlobalFilter
实现GlobalFilter, Ordered：
```java
@Component
@Slf4j
public class MyLogGateWayFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        log.info("come in MyLogGateWayFilter" + new Date());
        //do something
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return Integer.MIN_VALUE;
    }
}
```
## GatewayFilter Factories
### AddRequestHeader：
增加请求头
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback #断言,路径相匹配的进行路由
          filters:
            - AddRequestHeader=X-Request-red, blue #为请求增加请求头X-Request-red:blue
```
### AddRequestParameter：
增加请求参数
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback #断言,路径相匹配的进行路由
          filters:
            - AddRequestParameter=red, blue #为请求增加请求参数，例如:bytetrick.com/somePath?red=blue
```
### AddResponseHeader：
增加响应头
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback #断言,路径相匹配的进行路由
          filters:
            - AddResponseHeader=X-Response-Red, Blue #增加响应头 X-Response-Red:Blue
```
### DedupeResponseHeader：
响应头去重，RETAIN_FIRST: 默认值，保留第一个值,RETAIN_LAST: 保留最后一个值，RETAIN_UNIQUE: 保留所有唯一值，以它们第一次出现的顺序保留
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback #断言,路径相匹配的进行路由
          filters:
            - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin, RETAIN_FIRST
```
### Hystrix（1）：
使用myCommandName作为名称生成HystrixCommand 实例进行熔断管理，注意fallbackUri也需要在gateway配置route
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback #断言,路径相匹配的进行路由
          filters:
            - Hystrix=myCommandName
```
### Hystrix（2）：
Hystrix(2) 配置hystrix fallback后跳转uri，目前只支持 forward，注意fallbackUri也需要在gateway配置route，RewritePath按需配置，注意fallbackUri也需要在gateway配置route
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback #断言,路径相匹配的进行路由
          filters:
            - name: Hystrix
              args:
                name: fallbackcmd
                fallbackUri: forward:/payment/fallback
            - RewritePath=/payment/lb/**, /payment/fallback
```
### CircuitBreaker断路器：
断路器，注意fallbackUri也需要在gateway配置route
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback #断言,路径相匹配的进行路由
          filters:
            - name: CircuitBreaker
              args:
                name: myCircuitBreaker
                fallbackUri: forward:/payment/fallback
                #使用状态码触发断路器,例如在自定义globalFilter中exchange.getResponse().setStatusCode(HttpStatus.NOT_FOUND);
                statusCodes:
                  - 500
                  - "NOT_FOUND"
```
### FallbackHeaders：
为fallback添加hystrix降级或断路器的异常信息到请求头，需配置在fallback路由中
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback #断言,路径相匹配的进行路由
          filters:
            - name: FallbackHeaders
              args:
                executionExceptionTypeHeaderName: error-hearder
```
### MapRequestHeader：
将请求头Blue替换为X-Request-Red，value不变，如果X-Request-Red已存在则将Blue的value添加至之
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback #断言,路径相匹配的进行路由
          filters:
            - MapRequestHeader=Blue, X-Request-Red
```
### PrefixPath：
为请求添加前缀，如请求原来为/hello，则经过过滤器后为：/mypath/hello
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback #断言,路径相匹配的进行路由
          filters:
            - PrefixPath=/mypath
```
### PreserveHostHeader：
保留原有的hostHeader
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback #断言,路径相匹配的进行路由
          filters:
            - PreserveHostHeader
```
### RequestRateLimiter：
限流过滤器，Gateway内置redis限流器使用令牌桶算法，需要引入spring-data-redis。
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback #断言,路径相匹配的进行路由
          filters:
            - name: RequestRateLimiter
          args:
            redis-rate-limiter.replenishRate: 10
            redis-rate-limiter.burstCapacity: 20
            redis-rate-limiter.requestedTokens: 1
```
配置解析器：
```java
@Bean
KeyResolver userKeyResolver() {
    return exchange -> Mono.just(exchange.getRequest().getQueryParams().getFirst("user"));
}
```
此示例定义了每个用户10个请求速率限制。允许出现20个突发请求，但是在下一秒内，只有10个请求可用。KeyResolver是一个简单的获取用户请求参数的工具(注意，不建议在生产中使用)。
或者可以自己定义限流过滤器，实现 RateLimiter 并在配置中指定：
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback 
          filters:
            - name: RequestRateLimiter
          args:
            rate-limiter: "#{@myRateLimiter}"
            key-resolver: "#{@userKeyResolver}"
```
### RedirectTo：
根据exchange中设置的状态码重定向，状态码必须为3xx！否则：java.lang.IllegalArgumentException: status must be a 3xx code, but was 403。
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback 
          filters:
            - RedirectTo=302, https://baidu.com
```
### RemoveRequestHeader：
移除指定的请求头
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback 
          filters:
            - RemoveRequestHeader=X-Request-Foo
```
### RemoveResponseHeader：
移除指定的响应头
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback 
          filters:
            - RemoveResponseHeader=X-Response-Foo
```
### RemoveRequestParameter：
移除指定的请求参数
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback 
          filters:
            - RemoveRequestParameter=red
```
### RewritePath：
重写path
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback 
          filters:
            - RewritePath=/red(?<segment>/?.*), $\{segment}
```
### RewriteLocationResponseHeader：
重写Location响应头，隐藏后端细节，例如，对于POST api.example.com/some/object/name的请求，Location -service.prod.example.net/v2/some/object/id的位置响应头值被重写为 api.example.com/some/object/id。
stripVersionMode（版本号去除模式）：
NEVER_STRIP: 永不去除，即使原始请求没有版本号。

AS_IN_REQUEST 如果原始请求没带版本号则去除。

ALWAYS_STRIP 一定去除，即使原始请求带了版本号。

hostValue ：如果提供了hostValue参数，则用于替换响应位置标头的host:port部分。如果没有提供，则使用主机请求头的值。
```java
      routes:
      - id: rewritelocationresponseheader_route
        uri: http://example.org
        filters:
        - RewriteLocationResponseHeader=AS_IN_REQUEST, Location, ,
```
### RewriteResponseHeader：
重写响应头，例如： /42?user=ford&password=omg!what&flag=true在经过下游服务返回后将被替换成 /42?user=ford&password=***&flag=true
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback 
          filters:
            - RewriteResponseHeader=X-Response-Red, , password=[^&]+, password=***
```
### SaveSession：
转发到下游服务之前保存session（强制调用WebSession::save）
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback 
          filters:
            - SaveSession
```
### SecureHeaders
安全头配置，可自行配置spring.cloud.gateway.filter.secure-headers
```java
spring:
  application:
    name: cloud-gateway
  cloud:
    gateway:
      filter:
        secure-headers:
          disable:
            xss-protection-header,x-frame-options
          strict-transport-security: max-age=631138519
```

![image.png](https://image.bytetrick.com/2020/11/image-2953e2546dd1417f9d912b330a3a1c73.png)
或spring.cloud.gateway.filter.secure-headers.disable配置关闭。
### SetPath：
替换path，如下在转发给下游服务前 /red/blue 会被替换为 /blue
```java
      routes:
      - id: setpath_route
        uri: https://example.org
        predicates:
        - Path=/red/{segment}
        filters:
        - SetPath=/{segment}
```
### SetRequestHeader：
设置请求头，如果存在将会把值替换
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback 
          filters:
            - SetRequestHeader=X-Request-Red, Blue
```
或根据需要从path或host取值如下：
```java
      routes:
      - id: setrequestheader_route
        uri: https://example.org
        predicates:
        - Host: {segment}.myhost.org
        filters:
        - SetRequestHeader=foo, bar-{segment}
```
### SetRequestHeader：
设置响应头，如果存在将会把值替换
```java
      routes:
        - id: payment_route2
          uri: lb://CLOUD-PAYMENT-SERVICE
          predicates:
            - Path=/payment/testFallback 
          filters:
            - SetResponseHeader=X-Response-Red, Blue
```
或根据需要从path或host取值如下：
```java
      routes:
      - id: setrequestheader_route
        uri: https://example.org
        predicates:
        - Host: {segment}.myhost.org
        filters:
        - SetResponseHeader=foo, bar-{segment}
```
### SetStatus：
设置状态码，可以是int或枚举
```java
      routes:
      - id: setstatusstring_route
        uri: https://example.org
        filters:
        - SetStatus=BAD_REQUEST
```
也可以如此配置来返回原始状态码：
```java
spring:
  cloud:
    gateway:
      set-status:
        original-status-header-name: original-http-status
```
### StripPrefix：
去除一定数量的前缀，如下/name/blue/red 将会把“/name/blue”去除，转发到nameservice为：nameservice/red
```java
      routes:
      - id: nameRoot
        uri: https://nameservice
        predicates:
        - Path=/name/**
        filters:
        - StripPrefix=2
```
### Retry GatewayFilter
可配置重试策略：
retries: 需要重试次数.

statuses: 需要重试的状态码，org.springframework.http.HttpStatus.

methods: 需要重试的http方法，org.springframework.http.HttpMethod.

series: 需要重试的状态码系列，org.springframework.http.HttpStatus.Series.

exceptions: 重试的异常.

backoff: 重试间隔为firstBackoff * (factor ^ n)，n为第几次重试；如果配置了maxBackoff 则重试次数取决于maxBackoff；如果配置basedOnPreviousValue 为true，重试间隔为prevBackoff * factor
```java
      routes:
      - id: retry_test
        uri: http://localhost:8080/flakey
        predicates:
        - Host=*.retry.com
        filters:
        - name: Retry
          args:
            retries: 3
            statuses: BAD_GATEWAY
            methods: GET,POST
            backoff:
              firstBackoff: 10ms
              maxBackoff: 50ms
              factor: 2
              basedOnPreviousValue: false
```
### RequestSize GatewayFilter
限制请求数据大小，超过设定阈值则会返回413 Payload too large，可添加后周 KB,MB 默认为B。此过滤器默认配置为5MB
如下：
```java
      routes:
      - id: request_size_route
        uri: http://localhost:8080/upload
        predicates:
        - Path=/upload
        filters:
        - name: RequestSize
          args:
            maxSize: 5000000
```
如超过大小会返回errorMessage` : `Request size is larger than permissible limit. Request size is 6.0 MB where permissible limit is 5.0 MB
### SetRequestHost
替换host，在某些情况下需要替换HOST，如下间隔requst的host替换为example.org
```java
      routes:
      - id: set_request_host_header_route
        uri: http://localhost:8080/headers
        predicates:
        - Path=/headers
        filters:
        - name: SetRequestHost
          args:
            host: example.org
```
### ModifyRequestBody filter
使用此过滤器修改请求体中内容并传递给下游服务，如下请求传递user json经过滤器修改后传递给下游服务。
```java
    @Bean
    public RouteLocator testModifyRequestBodyFilter(RouteLocatorBuilder routeLocatorBuilder) {
        RouteLocatorBuilder.Builder routes = routeLocatorBuilder.routes();
        routes.route("", r -> r.path("/payment/testModifyRequestBodyFilter")
                .filters(f->f.preserveHostHeader().modifyRequestBody(String.class, User.class, MediaType.APPLICATION_JSON_VALUE,(exchange,s)->{
            User user = JSON.parseObject(s, User.class);
            user.setAge(new Random().nextInt(100));
            return Mono.just(user);
        })).uri("lb://CLOUD-PAYMENT-SERVICE"));
        return routes.build();
    }
```
![image.png](https://image.bytetrick.com/2020/11/image-e3ddb9dba6604ee0bd21fed5c30885b2.png)
### ModifyResponseBody filter
使用此过滤器修改响应体中内容并传递给上游，如下响应user json经过滤器修改后传递给上游。
```java
    @Bean
    public RouteLocator testModifyRequestBodyFilter(RouteLocatorBuilder routeLocatorBuilder) {
        RouteLocatorBuilder.Builder routes = routeLocatorBuilder.routes();
        routes.route("", r -> r.path("/payment/testModifyRequestBodyFilter")
                .filters(f->f.preserveHostHeader().modifyRequestBody(String.class, User.class, MediaType.APPLICATION_JSON_VALUE,(exchange,s)->{
            User user = JSON.parseObject(s, User.class);
            user.setAge(new Random().nextInt(100));
            return Mono.just(user);
        }).modifyResponseBody(String.class, User.class, MediaType.APPLICATION_JSON_VALUE,(exchange,s)->{
                    User user = JSON.parseObject(s, User.class);
                    user.setAge(new Random().nextInt(100));
                    return Mono.just(user);
                })).uri("lb://CLOUD-PAYMENT-SERVICE"));
        return routes.build();
    }
```
修改requestBody：
![image.png](https://image.bytetrick.com/2020/11/image-989a8536b95145fba4875734ed15296c.png)
修改responseBody：
![image.png](https://image.bytetrick.com/2020/11/image-c21784f4f92b43b995929c91c3eac1b0.png)
### Default Filters
如果需要给所有的route添加过滤器，可配置全局：
```java
spring:
  cloud:
    gateway:
      default-filters:
      - AddResponseHeader=X-Response-Default-Red, Default-Blue
      - PrefixPath=/httpbin
```
# 总结
以上便是spring cloud gateway 断言以及过滤器逐一介绍，gateway作为springcloud中新一代网关，使用netty作为底层容器，异步非阻塞对网关来说是比较合适的；值得注意的是使用gateway可大幅度提升网关吞吐量，但它并不会提升应用运行速度以及性能。