+++
title = "gateway监听nacos实现动态路由热更新"
author = "crzbird"
github_url = "https://github.com/crzbird"
head_img = ""
created_at = 2021-06-14T21:35:15
updated_at = 2021-06-14T21:35:15
description = "gateway监听nacos实现动态路由热更新"
tags = ["java","spring-cloud"]
+++
# nacos配置创建
将路由配置使用json格式单独创建
```json
[{
  "id": "dynamic-route",
  "order": 0,
  "predicates": [{
    "args": {
      "pattern": "/dynamic/**"
    },
    "name": "Path"
  }],
  "uri": "lb://come-service"
}]
```
# 编码配置gateway
## 路由配置类：根据nacos上路由配置
```java
/**
 * 路由配置类
 */
@Configuration
public class GateWayConfig {

    public static final long DEFAULT_TIMEOUT = 30000;

    public static String NACOS_SERVER_ADDR;

    public static String NACOS_NAMESPACE;

    public static String NACOS_ROUTE_DATA_ID;

    public static String NACOS_ROUTE_GROUP;

    @Value("${spring.cloud.nacos.config.server-addr}")
    public void setNacosServerAddr(String nacosServerAddr) {
        NACOS_SERVER_ADDR = nacosServerAddr;
    }

    @Value("${spring.cloud.nacos.config.namespace}")
    public void setNacosNamespace(String nacosNamespace) {
        NACOS_NAMESPACE = nacosNamespace;
    }

    @Value("${nacos.gateway.route.config.data-id}")
    public void setNacosRouteDataId(String nacosRouteDataId) {
        NACOS_ROUTE_DATA_ID = nacosRouteDataId;
    }

    @Value("${nacos.gateway.route.config.group}")
    public void setNacosRouteGroup(String nacosRouteGroup) {
        NACOS_ROUTE_GROUP = nacosRouteGroup;
    }
}
```
## 初始化路由
```java
/**
 *
 * 通过nacos下发动态路由配置,监听Nacos中gateway-route配置
 *
 */
@Component
@Slf4j
//@DependsOn({"gatewayConfig"}) // 依赖于gatewayConfig bean
public class DynamicRouteServiceImplByNacos {

    @Autowired
    private DynamicRouteServiceImpl dynamicRouteService;


    private ConfigService configService;

    @PostConstruct
    public void init() {
        log.info("gateway route init...");
        try{
            configService = initConfigService();
            if(configService == null){
                log.warn("initConfigService fail");
                return;
            }
            String configInfo = configService.getConfig(GateWayConfig.NACOS_ROUTE_DATA_ID, GateWayConfig.NACOS_ROUTE_GROUP, GateWayConfig.DEFAULT_TIMEOUT);
            log.info("获取网关当前配置:\r\n{}",configInfo);
            List<RouteDefinition> definitionList = JSON.parseArray(configInfo, RouteDefinition.class);
            for(RouteDefinition definition : definitionList){
                log.info("update route : {}",definition.toString());
                dynamicRouteService.add(definition);
            }
        } catch (Exception e) {
            log.error("初始化网关路由时发生错误",e);
        }
        dynamicRouteByNacosListener(GateWayConfig.NACOS_ROUTE_DATA_ID,GateWayConfig.NACOS_ROUTE_GROUP);
    }

    /**
     * 监听Nacos下发的动态路由配置
     * @param dataId
     * @param group
     */
    public void dynamicRouteByNacosListener (String dataId, String group){
        try {
            configService.addListener(dataId, group, new Listener()  {
                @Override
                public void receiveConfigInfo(String configInfo) {
                    log.info("进行网关更新:\n\r{}",configInfo);
                    List<RouteDefinition> definitionList = JSON.parseArray(configInfo, RouteDefinition.class);
                    log.info("update route : {}",definitionList.toString());
                    dynamicRouteService.updateList(definitionList);
                }
                @Override
                public Executor getExecutor() {
                    log.info("getExecutor\n\r");
                    return null;
                }
            });
        } catch (NacosException e) {
            log.error("从nacos接收动态路由配置出错!!!",e);
        }
    }

    /**
     * 初始化网关路由 nacos config
     * @return
     */
    private ConfigService initConfigService(){
        try{
            Properties properties = new Properties();
            properties.setProperty("serverAddr",GateWayConfig.NACOS_SERVER_ADDR);
            properties.setProperty("namespace",GateWayConfig.NACOS_NAMESPACE);
            return configService= NacosFactory.createConfigService(properties);
        } catch (Exception e) {
            log.error("初始化网关路由时发生错误",e);
            return null;
        }
    }
}
```
## 刷新最新的动态路由变化，实现动态增删改路由

```java
@Component
@Slf4j
@DependsOn("gateWayConfig")
public class DynamicRouteServiceImpl implements ApplicationEventPublisherAware {
    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;
    @Autowired
    private RouteDefinitionLocator routeDefinitionLocator;

    /**
     * 发布事件
     */
    @Autowired
    private ApplicationEventPublisher publisher;

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.publisher = applicationEventPublisher;
    }

    /**
     * 删除路由
     * @param id
     * @return
     */
    public String delete(String id) {
        try {
            log.info("gateway delete route id {}",id);
            this.routeDefinitionWriter.delete(Mono.just(id)).subscribe();
            this.publisher.publishEvent(new RefreshRoutesEvent(this));
            return "delete success";
        } catch (Exception e) {
            return "delete fail";
        }
    }

    /**
     * 更新路由
     * @param definitions
     * @return
     */
    public String updateList(List<RouteDefinition> definitions) {
        log.info("gateway update route {}",definitions);
        // 删除缓存routerDefinition
        List<RouteDefinition> routeDefinitionsExits =  routeDefinitionLocator.getRouteDefinitions().buffer().blockFirst();
        if (!CollectionUtils.isEmpty(routeDefinitionsExits)) {
            routeDefinitionsExits.forEach(routeDefinition -> {
                log.info("delete routeDefinition:{}", routeDefinition);
                delete(routeDefinition.getId());
            });
        }
        definitions.forEach(definition -> {
            updateById(definition);
        });
        return "success";
    }

    /**
     * 更新路由
     * @param definition
     * @return
     */
    public String updateById(RouteDefinition definition) {
        try {
            log.info("gateway update route {}",definition);
            this.routeDefinitionWriter.delete(Mono.just(definition.getId()));
        } catch (Exception e) {
            return "update fail,not find route  routeId: "+definition.getId();
        }
        try {
            routeDefinitionWriter.save(Mono.just(definition)).subscribe();
            this.publisher.publishEvent(new RefreshRoutesEvent(this));
            return "success";
        } catch (Exception e) {
            return "update route fail";
        }
    }

    /**
     * 增加路由
     * @param definition
     * @return
     */
    public String add(RouteDefinition definition) {
        log.info("gateway add route {}",definition);
        routeDefinitionWriter.save(Mono.just(definition)).subscribe();
        this.publisher.publishEvent(new RefreshRoutesEvent(this));
        return "success";
    }
}
```
# 至此实现动态路由热更新
注意：如果是在配置文件中配置的路由不会变动。
