---
layout: post
title: Spring Boot  Web项目(Freemarker)如何调试
date:   2021-08-18 11:11
description: Spring Boot  Web项目调试方式(Idea IDE)
categories: spring
comments: true
tags:
- spring
---

使用Spring Boot开发WEB项目时, 如果使用了Freemarker/jsp等模版文件, 那么我们需要:
* Java代码动态打断点
* 实时动态刷新模版文件的修改, 不用每次重启


### 1. 使用Maven插件: spring-boot:run
   spring-boot:run -Dspring-boot.run.fork=false
   fork参数设置为false, 这样在Java中才能打断点.
   原理: 如果fork了新进程, 则ide自动设置的远程调试端口绑定到了父进程, 则无法和新进程绑定, 所以就无法打断点

    此时Spring boot 的 devtools会自动被禁止, 所以就无法使用devtools的特性了.

![设置](/img/ideaide/maven-springboot-run.jpg )

### 2. 调试模式下启动 YourApplication(Spring Boot生成的Application)
   如果是Idea IDE, 设置 working directory 为: %MODULE_WORKING_DIR% , 否则jsp/freemarker功能无法找到页面文件.
   (原理可能需要读源码, 暂时不去研究)

    此模式支持devtools, 只要你重新build 模块, 应用会自动重新重启.

![设置](/img/ideaide/debug_springboot.jpg )

还可以研究一下 "Running Application Update Policies".


这样看起来第二种更简单, 就是有点奇怪.


### POM设置
包含devtools, tomcat(provided). 默认使用内嵌tomcat调试.

    <dependencies>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-freemarker</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
        </dependency>


        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <scope>provided</scope>
        </dependency>


    </dependencies>
