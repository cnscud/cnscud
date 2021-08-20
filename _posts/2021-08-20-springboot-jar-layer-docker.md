---
layout: post
title: Spring Boot使用Docker分层打包
date:   2021-08-20 20:11
description: docker中jar和war的分层打包
categories: spring
comments: true
tags:
- spring
---

Spring Boot 现在支持分层打包技术了, 我们也来用一用, 加速Docker打包, 构建的时候速度也会非常快.

## 分层设置 
首先pom里面要类似设置:

```xml
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <version>${spring-boot.version}</version>
                    <configuration>
                        <!-- 启用分层打包支持 -->
                        <layers>
                            <enabled>true</enabled>
                        </layers>
                    </configuration>
                    <executions>
                        <execution>
                            <goals>
                                <goal>repackage</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>



```

声明了 spring-boot-maven-plugin插件, 设置了layers配置, 开启分层支持.

打包完毕后, 我们检查jar包或者war包, 会发现多了一个 layers.idx文件, 里面包含了分层文件列表
```text
- "dependencies":
  - "WEB-INF/lib-provided/"
  - "WEB-INF/lib/HikariCP-4.0.3.jar"
  - "WEB-INF/lib/aspectjweaver-1.9.5.jar"
  ...
  ...
- "spring-boot-loader":
  - "org/"
- "snapshot-dependencies":
  - "WEB-INF/lib/ms-fundmain-base-1.0-SNAPSHOT.jar"
  - "WEB-INF/lib/xpower-main-1.0.3-SNAPSHOT.jar"
  - "WEB-INF/lib/xpower-utils-1.0.3-SNAPSHOT.jar"
- "application":
  - "META-INF/"
  - "WEB-INF/classes/"
  - "WEB-INF/jetty-web.xml"
  - "WEB-INF/layers.idx"
  - "pages/"
  - "static/"

```

此文件就是下面分层设置的依据.


如果是jar里面还有个classpath.idx文件, 里面列出了所有依赖的jar包.


打包的时候我们可以使用docker build 或者使用  docker-maven-plugin 插件来实现.

## 注意: spring-boot-maven-plugin 插件
本身就有docker打包功能, 不过下载打包速度太慢, 非常感人, 所有这里就不推荐了. --- 好处就是不用写Dockerfile, 简单方便, 缺点就是不能定制Docker文件.
配置类似如下:

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <image>
            <!--配置镜像名称-->
            <name>127.0.0.1:5000/springcnscud/${project.name}:${project.version}</name>
            <!--镜像打包完成后自动推送到镜像仓库-->
            <publish>true</publish>
        </image>
        <docker>
            <!--Docker远程管理地址-->
            <host>http://127.0.0.1:2375</host>
            <!-- 不使用TLS访问-->
            <tlsVerify>false</tlsVerify>
            <!--  Docker推送镜像仓库配置-->
            <publishRegistry>
                <!--推送镜像仓库用户名-->
                <username>cnscud</username>
                <!--推送镜像仓库密码-->
                <password>123456</password>
                <!--推送镜像仓库地址-->
                <url>http://127.0.0.1:5000</url>
            </publishRegistry>
        </docker>
    </configuration>
</plugin>

```

## 如果使用 docker-maven-plugin + 自定义Dockerfile的方式: 

pom配置: 
```xml
                <plugin>
                    <groupId>io.fabric8</groupId>
                    <artifactId>docker-maven-plugin</artifactId>
                    <version>${docker.plugin.version}</version>
                    <configuration>
                        <!-- Docker Remote Api-->
                        <!-- 本机则可以注释掉, 如果没有监听2375端口 -->
                        <dockerHost>${docker.host}</dockerHost>
                        <!-- Docker 镜像私服-->
                        <registry>${docker.registry}</registry>

                        <images>
                            <image>
                                <name>${docker.registry}/${docker.namespace}/${project.name}:${project.version}</name>
                                <build>
                                    <dockerFileDir>${project.basedir}</dockerFileDir>
                                </build>
                            </image>
                        </images>
                    </configuration>
                </plugin>
```

## 我们来看看Spring Boot的jar方式下的Dockerfile格式:
```dockerfile
# 分层构建, 加速增量构建

FROM adoptopenjdk/openjdk8:centos-slim as builder

WORKDIR application
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
RUN java -Djarmode=layertools -jar app.jar extract && rm app.jar

FROM adoptopenjdk/openjdk8:centos-slim

LABEL maintainer="cnscud@gmail.com"

ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
ENV JAVA_OPTS="-Xms128m -Xmx256m"

WORKDIR application

COPY --from=builder /application/dependencies/ ./
COPY --from=builder /application/snapshot-dependencies/ ./
COPY --from=builder /application/spring-boot-loader/ ./
COPY --from=builder /application/application/ ./

EXPOSE 9001

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.JarLauncher"]

```

里面的jdk请根据自己的情况修改, jar的情况下使用 JarLauncher.

## 如果是war怎么设置哪?

首先注意, 如果要独立运行, 可以使用嵌入式tomcat或jetty, pom里不要设置provider

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </dependency>
```

这样打包的时候就会包含嵌入式tomcat.

Dockerfile设置如下:

```dockerfile
# 分层构建, 加速增量构建

FROM adoptopenjdk/openjdk8:centos-slim as builder

WORKDIR application
ARG JAR_FILE=target/*.war
COPY ${JAR_FILE} app.war
RUN java -Djarmode=layertools -jar app.war extract && rm app.war

FROM adoptopenjdk/openjdk8:centos-slim
LABEL maintainer="cnscud@gmail.com"

ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
ENV JAVA_OPTS="-Xms128m -Xmx256m"

WORKDIR application

COPY --from=builder /application/dependencies/ ./
COPY --from=builder /application/snapshot-dependencies/ ./
COPY --from=builder /application/spring-boot-loader/ ./
COPY --from=builder /application/application/ ./

EXPOSE 8000

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.WarLauncher"]

```

注意文件名, 运行使用 WarLauncher.

## 使用外部tomcat 
未经实验, 构建分层可能比较麻烦...不过理论上也可以, 就是使用解压过的war包,而不是让tomcat自己解压

这里就不尝试了, 主要要点就是基础包换成tomcat, 运行的ENTRYPOINT换成tomcat, 中间把文件复制到容器里.

```dockerfile

FROM tomcat:9.0

#将target下的xx.war拷贝到/usr/local/tomcat/webapps/下
ADD ./target/xx.war /usr/local/tomcat/webapps/

#端口
EXPOSE 8080

#设置启动命令
ENTRYPOINT ["/usr/local/tomcat/bin/catalina.sh","run"]
```
