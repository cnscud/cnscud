---
layout: post 
title: Spring Cloud分区发布实践(1) 环境准备
date:   2021-06-07 14:07 
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
* [Spring Cloud分区发布实践(6) 根据Header选择实例区域](/springcloud/2021/06/07/springcloud-betazonedemo-6.html)


最近研究了一下Spring Cloud里面的灰度发布, 看到各种各样的使用方式, 真是纷繁复杂, 眼花缭乱, 不同的场景需要不同的解决思路.

那我们也来实践一下最简单的场景:

## 区域划分: 
服务分为beta(线上预发布环境)和main主生产环境

## 区域隔离情况
试情况可能有三种选择:
- A. main和beta互不相通, 绝对隔离 (资源相对充裕)
- B. main和beta正常情况下不通, 缺少实例时互通 (比较简单, 但可能无法区分异常服务, 不知道访问的是那个区域)
- C. beta绝对隔离, main在发布过程中可以切换到beta (main区域只有一台机器, 资源比较紧张)
- D. 发布前标注一套区域为beta的服务, 测试通过后修改beta服务的区域为main, 多个实例都可上线服务 (资源非常紧张, 操作相对复杂)


以上4种情况适合不同的公司, 但也各有利弊.

## 实现分析
我们先从Spring Cloud框架的技术方面来考虑
* B 最简单, 简单配置即可实现
* A, C 自己定制一下ServiceInstanceListSupplier 可以实现
* D 和A一样, 只不过需要额外修改服务的区域设置


## 项目环境
* Java 8 (1.8.0_291, build低版本有个bug)
* Spring Cloud 2020.0.3
* Spring Boot 2.5.0
* Intellij Idea 2021
* Maven 项目

## 项目目标
* 使用Spring Cloud开发微服务
* 支持2个区域: beta, main
* 服务链保持相同区域: gateway -> Service A -> Service B
* 使用Feign调用
* 使用Spring Cloud Gateway, LoadBalancer

## 创建根项目
首先我们在IDE里创建一个空的Maven项目, 把项目所有模块的公共依赖放在pom.xml

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.cnscud.betazone</groupId>
	<artifactId>betazone-root</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>pom</packaging>
	<description>hello</description>
	<url>https://www.cnscud.com</url>
	<modules>
	</modules>
	<properties>
		<spring-boot.version>2.5.0</spring-boot.version>
		<spring-cloud.version>2020.0.3</spring-cloud.version>

		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<maven.compiler.source>1.8</maven.compiler.source>
		<maven.compiler.target>1.8</maven.compiler.target>
	</properties>

	<dependencies>
		<!--监控 -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
		<!--Lombok -->
		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<scope>provided</scope>
		</dependency>
		<!--测试依赖 -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<dependencyManagement>
		<dependencies>
			<!-- spring boot 依赖 -->
			<dependency>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-dependencies</artifactId>
				<version>${spring-boot.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
			<!-- spring cloud 依赖 -->
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>


		</dependencies>
	</dependencyManagement>


</project>
```

## 准备注册中心

然后我们来启动一个Eureka作为我们的服务注册中心.

新建一个Module, 名字为 eureka-server, 使用IDE的Spring Initializr, 也可以使用 https://start.spring.io/ 创建.

![Spring Initializr](/img/springcloud/newmodule.jpg )

其中pom我们做一下修改, 继承父项目的pom:

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

    <artifactId>eureka-server</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>eureka-server</name>
    <description>Demo project for Spring Boot</description>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>

    </dependencies>
    
</project>
```

里面spring-cloud-starter-netflix-eureka-server 只是为了启动Eureka Server, 其他都不需要.

为了启动, 我们还需要配置一个application.yml:

```yaml
# 指定运行端口
server:
  port: 8001

# 指定服务名称
spring:
  application:
    name: eureka-server

# 指定主机地址
eureka:
  instance:
    hostname: localhost
  client:
    # 指定是否从注册中心获取服务(注册中心不需要开启)
    fetch-registry: false
    # 指定是否将服务注册到注册中心(注册中心不需要开启)
    register-with-eureka: false

```

然后就可以启动了, 访问 http://localhost:8001/ 就可以看到WEB界面, http://localhost:8001/eureka/apps 可以看到详细的xml内容, 可以用来验证metadata是否设置正确.


然后接下来我们准备用于测试的微服务...
