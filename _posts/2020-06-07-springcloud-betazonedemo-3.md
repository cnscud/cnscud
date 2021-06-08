---
layout: post 
title: Spring Cloud分区发布实践(3)
date:   2021-06-07 17:35 
description: Spring Cloud分区发布实践 
categories: springcloud 
comments: true 
tags:
- Spring, Spring Cloud, Zone

---
章节:
* [Spring Cloud分区发布实践(1)](/springcloud/2021/06/07/springcloud-betazonedemo-1.html)
* [Spring Cloud分区发布实践(2)](/springcloud/2021/06/07/springcloud-betazonedemo-2.html)
* [Spring Cloud分区发布实践(3)](/springcloud/2021/06/07/springcloud-betazonedemo-3.html)
* [Spring Cloud分区发布实践(4)](/springcloud/2021/06/07/springcloud-betazonedemo-4.html)


**注意: 因为涉及到配置测试切换, 中间环节需按此文章操作体验, 代码仓库里面的只有最后一步的代码**

准备好了微服务, 那我们就来看看网关+负载均衡如何一起工作

新建一个模块hello-gateway, 开启gateway和loadbalancer, pom部分如下:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.cnscud.betazone</groupId>
        <artifactId>betazone-root</artifactId>
        <version>0.0.1-SNAPSHOT</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <artifactId>hello-gateway</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>hello-gateway</name>
    <description>Demo project for Spring Boot</description>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-loadbalancer</artifactId>
        </dependency>

    </dependencies>


    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

HelloGatewayApplication 不修改, 使用默认的实现.


下面我们来实验几种配置, 看看LoadBalancer的表现

## 1. 默认配置, 不配置zone: application.yml (没有CustomLoadBalancerConfiguration标注)

```yaml
server:
  port: 8800

spring:
  application:
    name: betazone-hello-gateway
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

```

启动应用, 访问 http://localhost:8800/remoteapi/remote/id/2, 刷新几次浏览器, 可以看到内容在 

"World [remotename: 127.0.0.1:9001]"
"World [remotename: 127.0.0.1:9002]"

两种情况下依次切换, 说明LoadBalancer在生效, 但是没有任何zone设置.

## 2. 实验: 设置应用的metadata的zone, 负载均衡会不会自动感知哪?

我们复制一份application-main.yml, 修改如下:
```yaml
server:
  port: 8801

spring:
  application:
    name: betazone-hello-gateway
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
    metadata-map:
      zone: main
  client:
    register-with-eureka: true
    fetch-registry: true
    prefer-same-zone-eureka: true
    service-url:
      defaultZone: http://localhost:8001/eureka/


logging:
  level:
    org.springframework.cloud: debug

```

修改了eureka.instance.metadata-map.zone 为main, 端口改为8801, 按此配置启动应用, 访问 http://localhost:8801/remoteapi/remote/id/2

发现和第一种情况一样, zone没有生效, 说明默认的LoadBalancer没有考虑zone配置.

## 3. 开启LoadBalancer的zone感应 (main区域)
修改application-main.yml中loadbalancer部分如下:
```yaml
spring:
  cloud:
    loadbalancer:
      ribbon:
        enabled: false
      zone: main
      configurations: zone-preference
```

重新运行, 访问 http://localhost:8801/remoteapi/remote/id/2

现在生效了, 服务总是返回 "World [remotename: 127.0.0.1:9001]" , 不再返回端口 9002的信息了, 说明 zone区域设置生效了.

设置关键点:
* zone 设置loadbalancer要访问的服务的zone, 如果没设置, 则按需读取实例的eureka.instance.metadata-map.zone设置
* configurations 具体描述看Spring文档, 此处zone-preference 则设置优先按区域选择服务实例.

## 4. 测试LoadBalancer的zone感应 (beta区域)
复制一份application-beta.yml, 修改端口为 8802, 相关的zone都改为beta区域, 
修改application-main.yml中loadbalancer部分如下:
```yaml
server:
  port: 8802

spring:
  cloud:
    loadbalancer:
      ribbon:
        enabled: false
      zone: beta
      configurations: zone-preference
```

复制配置, 修改profile为beta, 启动实例, 访问 http://localhost:8802/remoteapi/remote/id/2, 则可以看到总是输出 "World [remotename: 127.0.0.1:9002]", 说明beta区域设置生效了.

此时我们可以看到如何使用内置的 "zone-preference"了, 那么我们如何自己定义个性化的策略哪, 我们可以看到Spring的这个配置指向 LoadBalancerClientConfiguration, 打开源码, 
可以看到里面有好多的discoveryClientServiceInstanceListSupplier实现, 我们自己实现(抄袭)一个不就可以了吗?

## 5. 定制ServiceInstanceListSupplier

我们先直接抄袭一份出来, 作为我们自己的实现(个性化实现代码不同, 但是流程一样, 这里主要跑通流程)

```java
/**
 * 自定义 Instance List Supplier: 根据默认Zone划分.
 * 也可以自己实现zone规则
 *
 */
public class CustomLoadBalancerConfiguration {

    @Bean
    public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                .withDiscoveryClient()
                .withZonePreference()
                .withCaching()
                .build(context);
    }

}
```

这里直接就是从 zonePreferenceDiscoveryClientServiceInstanceListSupplier 抄袭过来的, 我们主要先演示一下流程, 后面我们会实现自己的ServiceInstanceListSupplier.

声明配置以便生效: 
```java
@Configuration(proxyBeanMethods = false)
@LoadBalancerClients(defaultConfiguration = CustomLoadBalancerConfiguration.class)
public class SamezoneAutoConfiguration {
}

```

注释掉yml里面的 configurations: zone-preference, 其他不修改, 重新启动8801, 8802端口, 重新测试.

效果和上面一样, 说明通过类的方式也同样效果, 只不过我们可以使用自己定制的ServiceInstanceListSupplier 了.

* 注意:
**zone设置优先以 eureka.instance.metadata-map.zone  为首要设置, 没有特殊情况下不要设置spring.cloud.loadbalancer.zone, 这样loadbalancer就会读取实例的zone.**
  


代码位置: https://github.com/cnscud/javaroom/tree/main/betazone2/hello-gateway  注意代码只是最后一步的情况


接下来我们实验一下连锁调用的情况下, zone配置会不会被继承传递哪? 什么情况下会失效哪? ......
