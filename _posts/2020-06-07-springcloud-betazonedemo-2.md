---
layout: post 
title: Spring Cloud分区发布实践(2)
date:   2021-06-07 17:35 
description: Spring Cloud分区发布实践 
categories: springcloud 
comments: true 
tags:
- Spring, Spring Cloud, Zone

---
我们准备一下用于查询姓名的微服务.

首先定义一下服务的接口, 新建一个空的Maven模块hello-remotename-core, 里面新建一个类: 
```java
public interface RemoteNameService {

    String readName(int id) ;
}

```
接下来的微服务都实现这个简单的接口作为示范.


然后创建一个服务模块hello-remotename, 依然使用 Spring Initializr, 选择 "Spring Web", "Eureka Discovery Client" 2个模块.

其中的pom文件如下:
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

    <artifactId>hello-remotename</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>hello-remotename</name>
    <description>Demo project for Spring Boot</description>


    <dependencies>
        <dependency>
            <groupId>com.cnscud.betazone</groupId>
            <artifactId>hello-remotename-core</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>


    </dependencies>



</project>

```

此模块依赖接口模块, 用于实现接口. 然后我们实现一个服务的Controller, 如下:
```java
package com.cnscud.betazone.helloremotename.controller;

import com.cnscud.betazone.helloremotename.core.service.RemoteNameService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.env.Environment;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.net.Inet4Address;
import java.net.InetAddress;
import java.net.UnknownHostException;
import java.util.HashMap;
import java.util.Map;


@RestController
@RequestMapping("remote")
public class RemoteNameServiceController implements RemoteNameService {
    private static Logger logger = LoggerFactory.getLogger(RemoteNameServiceController.class);

    @Autowired
    Environment environment;

    final static String defaultName = "guest";
    static Map<Integer, String> names = new HashMap<>();

    static {
        names.put(1, "Felix");
        names.put(2, "World");
        names.put(3, "Sea");
        names.put(4, "Sky");
        names.put(5, "Mountain");
    }

    @Override
    @RequestMapping("/id/{id}")
    public String readName(@PathVariable("id") int id) {

        if( names.get(id) == null ) {
            return defaultName + getServerName();
        }
        else
        {
            return names.get(id) + getServerName();
        }
    }
    
    public String readServicePort() {
        return environment.getProperty("local.server.port");
    }

    public String readServiceIp() {
        InetAddress localHost = null;
        try {
            localHost = Inet4Address.getLocalHost();
        }
        catch (UnknownHostException e) {
            logger.error(e.getMessage(), e);
        }

        return localHost.getHostAddress();  // 返回格式为：xxx.xxx.xxx
    }

    public String getServerName() {
        return " [remotename: " + readServiceIp() + ":" + readServicePort() + "]";
    }

}

```

RemoteNameServiceController实现了RemoteNameService 接口, 为了后续方便区分是哪个实例在服务, 返回的信息里增加了IP和端口信息.

然后声明application.yml, 在9001端口启动

```yaml
server:
  port: 9001


spring:
  application:
    name: betazone-hello-remotename

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

可以看到, 此实例端口为9001, 服务区域zone设置为 main.

然后在复制一个为 application-beta.yml, 修改如下
```yaml
server:
  port: 9002

spring:
  application:
    name: betazone-hello-remotename

eureka:
  instance:
    prefer-ip-address: true
    metadata-map:
      zone: beta #服务区域
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://localhost:8001/eureka/

logging:
  level:
    org.springframework.cloud: debug

```
此实例设置端口为9002, 服务区域为 beta.

启动第一个Application, 然后复制配置, 修改profile为beta , 启动第二个实例.

此时去Eureka查看, 可以看到betazone-hello-remotename有2个服务, 使用xml查看 http://localhost:8001/eureka/apps , 可以看到不同的metadata.

![控制台](/img/springcloud/spring-services-dashboard.jpg ) 

点击访问 http://localhost:9001/remote/id/2 或 http://localhost:9002/remote/id/2 则可以看到我们刚刚运行的服务.

项目代码: https://github.com/cnscud/javaroom/tree/main/betazone2/hello-remotename

接下来我们看看使用gateway代理服务的效果...