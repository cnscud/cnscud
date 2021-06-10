---
layout: post 
title: Spring Cloud分区发布实践(6)--灰度服务-根据Header选择实例区域
date:   2021-06-07 17:35 
description: Spring Cloud分区发布实践 
categories: springcloud 
comments: true 
tags:
- Spring, Spring Cloud, Zone

---
章节:
* [Spring Cloud分区发布实践(1) 准备环境](/springcloud/2021/06/07/springcloud-betazonedemo-1.html)
* [Spring Cloud分区发布实践(2) 微服务](/springcloud/2021/06/07/springcloud-betazonedemo-2.html)
* [Spring Cloud分区发布实践(3) 网关和负载均衡](/springcloud/2021/06/07/springcloud-betazonedemo-3.html)
* [Spring Cloud分区发布实践(4) FeignClient](/springcloud/2021/06/07/springcloud-betazonedemo-4.html)
* [Spring Cloud分区发布实践(5) 定制ServiceInstanceListSupplier](/springcloud/2021/06/07/springcloud-betazonedemo-5.html)
* [Spring Cloud分区发布实践(6) 灰度服务-根据Header选择实例区域](/springcloud/2021/06/07/springcloud-betazonedemo-6.html)


此文是一个完整的例子, 包含可运行起来的源码.

![运行的实例列表](/img/springcloud/zonebyheader_running.jpg )

此例子包含以下部分:
* 网关层实现自定义LoadBalancer, 根据Header选取实例
* 服务中的Feign使用拦截器, 读取Header
* Feign的LoadBalancer也是用网关一样的实现
* 使用Web Filter来统一设置header变量, 于业务解耦

## 自定义LoadBalancer, 读取Header
首先创建一个新模块 hello-mybalancerbyheader, pom文件如下

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.cnscud.betazone</groupId>
        <artifactId>betazone-root</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>
    <artifactId>hello-mybalancerbyheader</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>hello-mybalancerbyheader</name>
    <description>Demo project for Spring Boot</description>

    <dependencies>

        <dependency>
            <groupId>com.cnscud.betazone</groupId>
            <artifactId>hello-pubtool</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-loadbalancer</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

    </dependencies>

</project>

```

创建一个 application.yml,

```yaml
server:
  port: 9200

spring:
  application:
    name: betazone-hello-mybalancerbyheader
  main:
    allow-bean-definition-overriding: true
  cloud:
    gateway:
      discovery:
        locator:
          lowerCaseServiceId: true
          enabled: true
      routes:
        - id: remotename
          uri: lb://betazone-hello-remotename
          predicates:
            - Path=/remoteapi/**
          filters:
            - StripPrefix=1
    loadbalancer:
      ribbon:
        enabled: false



eureka:
  instance:
    prefer-ip-address: true

  client:
    register-with-eureka: true
    fetch-registry: true
    prefer-same-zone-eureka: true
    service-url:
      defaultZone: http://localhost:8001/eureka/


logging:
  level:
    org.springframework.cloud: debug
    com.cnscud.betazone: debug


```

代理了后面的betazone-hello-remotename服务, 启动, 访问 <http://localhost:9200/remoteapi/remote/id/2>{:target="_blank"}  说明正常, 后面实例是不停轮询的方式来变化的.


那我们来如何根据header访问后面不同的实例哪? 方法有很多, 我们采用最简洁的办法, 抄袭一个Spring自己的 RoundRobinLoadBalancer, 😅

(代码放在hello-pubtool模块, 一会要复用)

```java
package com.cnscud.betazone.pub.zonebyheader;

import org.apache.commons.lang.StringUtils;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.loadbalancer.DefaultResponse;
import org.springframework.cloud.client.loadbalancer.EmptyResponse;
import org.springframework.cloud.client.loadbalancer.Request;
import org.springframework.cloud.client.loadbalancer.RequestDataContext;
import org.springframework.cloud.client.loadbalancer.Response;
import org.springframework.cloud.loadbalancer.core.NoopServiceInstanceListSupplier;
import org.springframework.cloud.loadbalancer.core.ReactorServiceInstanceLoadBalancer;
import org.springframework.cloud.loadbalancer.core.SelectedInstanceCallback;
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;
import org.springframework.http.HttpHeaders;
import reactor.core.publisher.Mono;

import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Random;
import java.util.Set;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * 根据header里面的workzone选择合适的实例, 如果没有发现, 则返回所有实例.
 *
 * A Round-Robin-based implementation of {@link ReactorServiceInstanceLoadBalancer}.
 *
 * @author Spencer Gibb
 * @author Olga Maciaszek-Sharma
 */
public class MyBetaMainByHeaderLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    private static final Log log = LogFactory.getLog(MyBetaMainByHeaderLoadBalancer.class);
    final String defaultKey = "default";

    final AtomicInteger position;
    final Map<String, AtomicInteger> postionMap = new HashMap<>();

    final String serviceId;

    ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider;

    /**
     * @param serviceInstanceListSupplierProvider a provider of
     * {@link ServiceInstanceListSupplier} that will be used to get available instances
     * @param serviceId id of the service for which to choose an instance
     */
    public MyBetaMainByHeaderLoadBalancer(ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider,
                                          String serviceId) {
        this(serviceInstanceListSupplierProvider, serviceId, new Random().nextInt(1000));
    }

    /**
     * @param serviceInstanceListSupplierProvider a provider of
     * {@link ServiceInstanceListSupplier} that will be used to get available instances
     * @param serviceId id of the service for which to choose an instance
     * @param seedPosition Round Robin element position marker
     */
    public MyBetaMainByHeaderLoadBalancer(ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider,
                                          String serviceId, int seedPosition) {
        this.serviceId = serviceId;
        this.serviceInstanceListSupplierProvider = serviceInstanceListSupplierProvider;
        this.position = new AtomicInteger(seedPosition);
        postionMap.put(defaultKey, this.position);
    }

    @SuppressWarnings("rawtypes")
    @Override
    // see original
    // https://github.com/Netflix/ocelli/blob/master/ocelli-core/
    // src/main/java/netflix/ocelli/loadbalancer/RoundRobinLoadBalancer.java
    public Mono<Response<ServiceInstance>> choose(Request request) {
        //read headers
        HttpHeaders headers = ((RequestDataContext) request.getContext()).getClientRequest().getHeaders();

        ServiceInstanceListSupplier supplier = serviceInstanceListSupplierProvider
                .getIfAvailable(NoopServiceInstanceListSupplier::new);
        return supplier.get(request).next()
                .map(serviceInstances -> processInstanceResponse(supplier, serviceInstances, headers));
    }

    private Response<ServiceInstance> processInstanceResponse(ServiceInstanceListSupplier supplier,
                                                              List<ServiceInstance> serviceInstances,
                                                              HttpHeaders headers ) {
        Response<ServiceInstance> serviceInstanceResponse = getInstanceResponse(serviceInstances, headers);
        if (supplier instanceof SelectedInstanceCallback && serviceInstanceResponse.hasServer()) {
            ((SelectedInstanceCallback) supplier).selectedServiceInstance(serviceInstanceResponse.getServer());
        }
        return serviceInstanceResponse;
    }

    private Response<ServiceInstance> getInstanceResponse(List<ServiceInstance> instances,
                                                          HttpHeaders headers) {
        if (instances.isEmpty()) {
            if (log.isWarnEnabled()) {
                log.warn("No servers available for service: " + serviceId);
            }
            return new EmptyResponse();
        }

        String workzone = headers.getFirst("workzone");
        log.info("getInstanceResponse: workzone-> " + workzone);
        String positionKey = defaultKey;

        Map<String,String> zoneMap = new HashMap<>();
        zoneMap.put("zone",workzone);
        final Set<Map.Entry<String,String>> attributes =
                Collections.unmodifiableSet(zoneMap.entrySet());

        List<ServiceInstance> lastInstanceList = instances;
        if(StringUtils.isNotBlank(workzone)) {
            lastInstanceList = new ArrayList<>();
            for (ServiceInstance instance : instances) {
                Map<String, String> metadata = instance.getMetadata();
                //根据zone头部判断
                if (metadata.entrySet().containsAll(attributes)) {
                    lastInstanceList.add(instance);
                }
            }

            //此处如果没有发现任何一个instance, 返回所有instance: 请根据自己情况定义
            if(lastInstanceList.size() <=0){
                lastInstanceList = instances;
            }
            else {
                positionKey = workzone;
            }
        }

        AtomicInteger mypos = postionMap.get(positionKey);
        if( mypos == null) {
            mypos = new AtomicInteger(new Random().nextInt(1000));
            postionMap.put(positionKey, mypos);
        }

        int pos = Math.abs(mypos.incrementAndGet());

        ServiceInstance instance = lastInstanceList.get(pos % lastInstanceList.size());

        return new DefaultResponse(instance);
    }

}

```

代码首先定义了一个postionMap, 用来存放不同区域的上次访问的位置, 避免每次都访问第一个实例. 然后获取Header 
HttpHeaders headers = ((RequestDataContext) request.getContext()).getClientRequest().getHeaders();

然后读取 "workzone" 字段, 如果存在, 则遍历instances, 发现实例的metadata的zone如果和当前字段一样, 则加到列表里.

检查完成, 使用命中的列表进行轮询...如果没有实例, 则使用所有实例进行轮询.

###实例化LoadBalancer(为了复用, 放在hello-pubtool模块里) 
```java

/**
 * 定义配置, 引入LoadBalancer.
 *
 * @author Felix Zhang 2021-06-09 16:46
 * @version 1.0.0
 */
public class MyBetaMainByHeaderLoadBalancerConfiguration {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> reactorServiceInstanceLoadBalancer(Environment environment,
                                                                                   LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new MyBetaMainByHeaderLoadBalancer(
                loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }

}
```

###声明配置(在hello-mybalancerbyheader模块里使用):
```java

/**
 * 声明配置类.
 */
@Configuration(proxyBeanMethods = false)
@LoadBalancerClients(defaultConfiguration = MyBetaMainByHeaderLoadBalancerConfiguration.class)
public class LoadBalancerAutoConfiguration {
}

```

启动应用, 我们来测试一下, 
```shell
localhost:cnscud.github.io felixzhang$ curl -H 'workzone:beta' "http://localhost:9200/remoteapi/remote/id/2"
World [remotename: 127.0.0.1:9002]

localhost:cnscud.github.io felixzhang$ curl -H 'workzone:beta' "http://localhost:9200/remoteapi/remote/id/2"
World [remotename: 127.0.0.1:9002]
 
```

发现返回的都是beta区域的实例, 说明代码正确.

##Feign实现根据Header访问不同区域的实例
第一步都是简单的, 我们看看Feign如何也能继承上一层的效果哪? 启用同样的LoadBalancer, 但是从哪读Header哪, 因为不是同一个Request, 显然是读不到的.

为了把代码区分出来, 不破坏原来的例子, 我们复制一个hello-nameservice模块到 hello-nameservicebyheader.

先打开上面一样的配置:
```java
/**
 * 配置声明类: .
 */
@Configuration(proxyBeanMethods = false)
@LoadBalancerClients(defaultConfiguration = MyBetaMainByHeaderLoadBalancerConfiguration.class)
public class LoadBalancerByHeaderAutoConfiguration {
}

```

但是这没什么用, 因为读不到Header.

然后我去网上搜了搜, 读了5*5=25篇文章之后.....中间过程省略50000字......

### 实现个Feign的拦截器

我们先看看原来的Controller代码,

```java
@RequestMapping("/id/{userid}")
    public String helloById(@PathVariable("userid") String userid, HttpServletRequest request) {
        logger.debug("call helloById with " + userid);

        logger.info("[nameservice] workzone header:" + request.getHeader("workzone"));

        if (StringUtils.isNotBlank(userid) && StringUtils.isNumeric(userid)) {
            return "hello " + feignRemoteNameService.readName(Integer.parseInt(userid)) + getServerName();
        }

        return "hello guest"  +  getServerName();
    }

```

里面是可以读取到request.getHeader("workzone")的, 但是进入到feign的服务里面, 显然就读不到了, 而Feign的拦截器可以帮我们把Header放进去.
所以我们首先找个地方把这个header存起来,

因为不知道服务什么情况下会被调用, 有可能是异步, 有可能是线程池, 有可能....总之, 此处省略10000字, 我们实现一个:

### 实现一个Holder, 存放变量
```java


/**
 * Holder: store a String.
 * !!!! 仅供参考, 没有全面测试过.
 *
 * 注意: 测试的几种情况要考虑: 父子线程, 线程池, 高QPS等等: InheritableThreadLocal, TransmittableThreadLocal, HystrixRequestVariableDefault
 *
 * @author Felix Zhang 2021-06-10 10:37
 * @version 1.0.0
 */
public class RequestHeaderHolder {
    //如果存Map, 好像有问题, 此处用String
    private static final ThreadLocal<String> MYLOCAL;

    static {
        MYLOCAL = new InheritableThreadLocal();
        //new TransmittableThreadLocal();

    }

    public static String get() {
        return MYLOCAL.get();
    }


    public static void set(String value) {
        MYLOCAL.set(value);
    }

    public static void remove() {
        MYLOCAL.remove();
    }


}

```

至于有没有问题, 因为不太好验证各种各种极限情况, 所以在实战中迎接战火吧, 总之也就InheritableThreadLocal/TransmittableThreadLocal/HystrixRequestVariableDefault这几个东西了.

然后我们在Controller把header存下来:
```java
    RequestHeaderHolder.set(request.getHeader("workzone"));
```

此处实现一个拦截器:
```java

/**
 * Feign Interceptor for transfer header.
 *
 * @author Felix Zhang 2021-06-10 10:04
 * @version 1.0.0
 */
public class FeignRequest4ZoneHeaderInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        //获取之前设置的header
        String workzone = RequestHeaderHolder.get();

        if(workzone !=null){
            template.header("workzone", workzone);
        }
    }
}

```

超级简单的拦截器, 把workzone这个header放到RequestTemplate的header里面去.

然后实例化拦截器
```java
/**
 * Interceptor configuration 声明拦截器配置.
 *
 * @author Felix Zhang 2021-06-10 11:09
 * @version 1.0.0
 */
public class FeignByZoneHeaderConfig {
    @Bean("myInterceptor")
    public RequestInterceptor getRequestInterceptor() {
        return new FeignRequest4ZoneHeaderInterceptor();
    }
}

```

启用配置(全局声明):
```java
@EnableFeignClients(basePackages = "com.cnscud.betazone.hellonameservicebyheader", defaultConfiguration = FeignByZoneHeaderConfig.class)
```

或者单独声明
```java
@FeignClient(value = "betazone-hello-remotename", configuration = FeignByZoneHeaderConfig.class)
```

hello-mybalancerbyheader模块里添加路由: 
```yaml
        - id: default
          uri: lb://betazone-hello-nameservicebyheader
          predicates:
            - Path=/api/**
          filters:
            - StripPrefix=1

```

重新启动应用, 访问  , 发现生效了
```shell
localhost:cnscud.github.io felixzhang$ curl -H 'workzone:beta' "http://localhost:9200/api/remote/id/2"
hello World [remotename: 127.0.0.1:9002] [nameservice:127.0.0.1:8203]

localhost:cnscud.github.io felixzhang$ curl -H 'workzone:beta' "http://localhost:9200/api/remote/id/2"
hello World [remotename: 127.0.0.1:9002] [nameservice:127.0.0.1:8203]

```

返回的一直是beta区域的实例.

### 来个Web Filter

我们回想一下, 刚才
```java
    RequestHeaderHolder.set(request.getHeader("workzone"));
```

是写在Controller里面的, 这显然不合适, 这个谁也看不懂, 和业务也没啥关系, 那我们可以放到Filter里面去, Filter就很容易了, 就是个普通的Web Filter就行.

在hello-pubtool模块里创建ZoneHeaderFilter, 为了别处公用:

```java

/**
 * Web Filter.
 *
 * @author Felix Zhang 2021-06-10 16:45
 * @version 1.0.0
 */
public class ZoneHeaderFilter  extends OncePerRequestFilter {

    private final Log logger = LogFactory.getLog(this.getClass());

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        logger.info("[ZoneHeaderFilter] workzone header:" + request.getHeader("workzone"));

        //给Feign Client用的, 使用位置: FeignRequest4ZoneHeaderInterceptor
        RequestHeaderHolder.set(request.getHeader("workzone"));

        filterChain.doFilter(request, response);
    }
    
}

```

这也太简单了....
在Feign的模块里声明 WebZoneHeaderFilterConfig: 
```java
/**
 * 注入Filter.
 *
 * @author Felix Zhang 2021-06-10 17:05
 * @version 1.0.0
 */
@Configuration
public class WebZoneHeaderFilterConfig {
    @Bean
    public FilterRegistrationBean<ZoneHeaderFilter> zoneheaderFilter() {
        FilterRegistrationBean<ZoneHeaderFilter> registrationBean
                = new FilterRegistrationBean<>();

        registrationBean.setFilter(new ZoneHeaderFilter());
        registrationBean.addUrlPatterns("/*");

        return registrationBean;
    }
}

```

收工了....

删除掉Controller里面的代码, 重新启动应用, 访问, 发现依然生效了
```shell
localhost:cnscud.github.io felixzhang$ curl -H 'workzone:beta' "http://localhost:9200/api/remote/id/2"
hello World [remotename: 127.0.0.1:9002] [nameservice:127.0.0.1:8203]

localhost:cnscud.github.io felixzhang$ curl -H 'workzone:beta' "http://localhost:9200/api/remote/id/2"
hello World [remotename: 127.0.0.1:9002] [nameservice:127.0.0.1:8203]

```

返回的一直是beta区域的实例.

收工.

## 感想:
网上的方法非常多, 概念非常多...可能是因为Spring Cloud的可定制性太强了, 组件太多了, 拦截器/Filter/各种配置/Rule/AOP/LoadBalancer各种都可以定制....
反观我的实现, 大部分是抄袭Spring自己的实现, 自己的代码没有几行, 这样我就放心多了...

## 注意
每个人的情况都不一样, 不同的组件, 不同的实现方式, 都可能造成不一样的效果, 所以仅供参考, 学到手才算真的好!


所有教程里的项目源码: <https://github.com/cnscud/javaroom/tree/main/betazone2>{:target="_blank"}

感谢网上的各种文章, 太多了, 就不一一贴出来了. (很多写的不全, 过时, 所以读的时候也很费劲, 知识爆炸的时代学习成本也很高, 选择多了也不见得都是好事, 当然也是好事)

##
感谢感谢, 感谢帮助我的朋友, 家人. 感谢你们. 2021.6.10