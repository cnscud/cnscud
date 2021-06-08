---
layout: post 
title: Spring Cloud分区发布实践(4)
date:   2021-06-07 17:35 
description: Spring Cloud分区发布实践 
categories: springcloud 
comments: true 
tags:
- Spring, Spring Cloud, Zone

---
上面看到直接通过网关访问微服务是可以实现按区域调用的, 那么微服务之间调用是否也能按区域划分哪?

下面我们使用FeignClient来调用微服务, 就可以配合LoadBalancer实现按区域调用.

首先我们新建一个微服务模块 hello-nameservice, 用来调用 hello-remotename服务. 模块需要使用Feign, 还要开启Feign的负载均衡, pom.xml文件如下:

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

    <artifactId>hello-nameservice</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>hello-nameservice</name>
    <description>Demo project for Spring Boot</description>

    <dependencies>
        <dependency>
            <groupId>com.cnscud.betazone</groupId>
            <artifactId>hello-remotename-core</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.apache.commons/commons-lang3 -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.0</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webflux</artifactId>
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

注意上面的 webflux , 用于新版本的DiscoveryClient适配.

## 使用Feign调用微服务

首先声明一个被调用服务的Feign接口, 如下

```java
@FeignClient(value = "betazone-hello-remotename")
public interface FeignRemoteNameService extends RemoteNameService {

    @RequestMapping("/remote/id/{id}")
    @Override
    String readName(@PathVariable("id") int id) ;
}

```

这个类映射到前面讲过的 "betazone-hello-remotename", 接口格式一致, 使用FeignClient标注.

然后我们实现自己的微服务逻辑:

```java
package com.cnscud.betazone.hellonameservice.feign;

import org.apache.commons.lang3.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.env.Environment;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.net.Inet4Address;
import java.net.InetAddress;
import java.net.UnknownHostException;

/**
 * Hello Controller from Remote Service.
 *
 * @author Felix Zhang 2021-06-04 09:29
 * @version 1.0.0
 */
@RestController
@RequestMapping("remote")
public class HelloNameByRemoteController {

    private static Logger logger = LoggerFactory.getLogger(HelloNameByRemoteController.class);

    @Autowired
    private FeignRemoteNameService feignRemoteNameService;

    @Autowired
    Environment environment;

    @RequestMapping("/id/{userid}")
    public String helloById(@PathVariable("userid") String userid) {
        logger.debug("call helloById with " + userid);

        if (StringUtils.isNotBlank(userid) && StringUtils.isNumeric(userid)) {
            return "hello " + feignRemoteNameService.readName(Integer.parseInt(userid)) + getServerName();
        }

        return "hello guest"  +  getServerName();
    }
    
    //......其他代码

}


```

这个类里面注入了FeignRemoteNameService服务, Feign会自动初始化.

为了让Feign能用, 我们还必须启用 @EnableFeignClients(basePackages = "com.cnscud.betazone.hellonameservice"), 包名就是你的服务的包名. 声明可以放在HelloNameServiceApplication 类里面.

准备一下应用的配置 application.yml

```yaml
server:
  port: 8101

spring:
  application:
    name: betazone-hello-nameservice
  cloud:
    loadbalancer:
      ribbon:
        enabled: false

eureka:
  instance:
    prefer-ip-address: true
    metadata-map:
      zone: main #服务区域
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://localhost:8001/eureka/

logging:
  level:
    org.springframework.cloud: debug

```

启动应用, 访问 http://localhost:8101/remote/id/2 , 正常情况下, 访问到的remotename服务是不确定的, 9001或者9002, 看来还是需要做一些设置才能按区域生效.

通用, 回想上一节的内容, 我们使用zone-preference 或者自定义ServiceInstanceListSupplier都可以实现, 这里不在重复.

代码里依然使用了 SamezoneAutoConfiguration 和CustomLoadBalancerConfiguration, 就可以按区域访问了.

我们在复制一份配置,用于beta区域, application-beta.yml 如下

```yaml
server:
  port: 8103

spring:
  application:
    name: betazone-hello-nameservice
  cloud:
    loadbalancer:
      ribbon:
        enabled: false

eureka:
  instance:
    prefer-ip-address: true
    metadata-map:
      zone: beta # zone服务区域 beta
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://localhost:8001/eureka/


logging:
  level:
    org.springframework.cloud: debug

```

这个实例用于接下来演示区域继承的beta区域用途.

## 网关里声明betazone-hello-nameservice 微服务

让我们回到之前的hello-gateway项目, 修改application.yml等三个文件, routes节点下变更为: 

```yaml
      routes:
        - id: default
          uri: lb://betazone-hello-nameservice
          predicates:
            - Path=/api/**
          filters:
            - StripPrefix=1
        - id: remotename
          uri: lb://betazone-hello-remotename
          predicates:
            - Path=/remoteapi/**
          filters:
            - StripPrefix=1

```

这样网关就代理了betazone-hello-nameservice服务, 路径为/api . 重新启动三个gateway应用的实例, 试着访问

* 默认实例(无区域设置) http://localhost:8800/api/remote/id/2 , 发现背后的2个微服务都在轮询.
* main实例(区域为main)  http://localhost:8801/api/remote/id/2 发现背后的2个微服务只调用了区域为main的实例, 区域可以继承.
* beta实例(区域为beta) http://localhost:8802/api/remote/id/2 发现背后的2个微服务只调用了区域为beta的实例, 区域可以继承.

到此, 我们的目的达到, 可以按区域来划分服务了.

**特殊备注: 如果某个微服务缺少某个区域的实例, 此项目用的ServiceInstanceListSupplier会自动使用所有实例, 那此时zone的继承就继承的是被使用的实例的zone了, 而不是网关的zone设置了.**

项目图如下: 

![架构图](/img/springcloud/callflow.png )

## 实际应用场景?
假设我们有个网站, 正常访问是 http://www.cnscud.com , 线上预发布时 http://beta.cnscud.com , 就可以设置两个区域, 通过Nginx映射, 就可以映射到不同的网关, 就自然可以区分了.


项目源码: https://github.com/cnscud/javaroom/tree/main/betazone2


接下来我们试试定制自己的自己定制一下ServiceInstanceListSupplier ......