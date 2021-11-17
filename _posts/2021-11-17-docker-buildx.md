---
layout: post
title: 使用buildx来构建支持多平台的Docker镜像(Mac系统)
date:   2021-11-17 16:09
description: 在Mac系统下, 使用buildx来构建支持多平台的Docker镜像
categories: docker
comments: true
tags:
- docker,buildx
---

最近换了Macbook Pro m1电脑, 系统的架构变成了arm64, 软件重新安装了一大堆, 发现运行docker拉取镜像, 很多
```shell
➜  ~ docker pull mysql:5.6
5.6: Pulling from library/mysql
5.6: Pulling from library/mysql
5.6: Pulling from library/mysql
no matching manifest for linux/arm64/v8 in the manifest list entries
```

看见没有, mysql 5.6不支持arm64架构, 所以docker就启动不了, 切换为mysql 8就可以了
```shell
➜  ~ docker pull mysql/mysql-server:8.0
8.0: Pulling from mysql/mysql-server
8.0: Pulling from mysql/mysql-server
Digest: sha256:91170bd4e012f0bf46b5141a38b612427b37692e8465cdffe9b0ca2d74d37d8a
Status: Downloaded newer image for mysql/mysql-server:8.0
docker.io/mysql/mysql-server:8.0
```

哪自己发布的镜像如何支持多平台的镜像哪? 常规的思路就是有多台电脑, 装不同的系统, 然后发布镜像.

我们可以为 docker 命令行安装 buildx 插件扩展其功能。buildx 能够使用由 Moby BuildKit 提供的构建镜像额外特性，它能够创建多个 builder 实例，在多个节点并行地执行构建任务，以及跨平台构建。

## 准备环境
macOS 或 Windows 系统的 Docker Desktop，以及 Linux 发行版通过 deb 或者 rpm 包所安装的 docker 内置了 buildx，不需要另行安装。

我的Macbook Pro系统版本是 MacOS Monterey 12.0.1, 安装的 Docker Desktop版本是 4.2.0 (70708). 默认是带了buildx的.

```shell
➜  ~ docker buildx  version
github.com/docker/buildx v0.6.3 266c0eac611d64fcc0c72d80206aa364e826758d
```

默认情况下是没有启用buildx配置的, 需要自己创建一个:
```shell
docker buildx create --name felixbuild
docker buildx use felixbuild
```
创建一个自己的构建器, 并检查确认:
```shell
➜  ~ docker buildx  ls
NAME/NODE       DRIVER/ENDPOINT             STATUS  PLATFORMS
felixbuild *    docker-container
  felixbuild0   unix:///var/run/docker.sock running linux/arm64, linux/amd64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/mips64le, linux/mips64, linux/arm/v7, linux/arm/v6
desktop-linux   docker
  desktop-linux desktop-linux               running linux/arm64, linux/amd64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
default         docker
  default       default                     running linux/arm64, linux/amd64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
➜  ~ docker buildx inspect
Name:   felixbuild
Driver: docker-container

Nodes:
Name:      felixbuild0
Endpoint:  unix:///var/run/docker.sock
Status:    running
Platforms: linux/arm64, linux/amd64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/mips64le, linux/mips64, linux/arm/v7, linux/arm/v6
```

可以看到默认启用了 felixbuild构建器.

查看docker的进程, 发现启动了一个容器用来构建:
```shell
➜  ~ docker ps
CONTAINER ID   IMAGE                             COMMAND          CREATED             STATUS             PORTS     NAMES
e32a75df8499   moby/buildkit:buildx-stable-1     "buildkitd"      About an hour ago   Up About an hour             buildx_buildkit_felixbuild0
```

这个进程不要删除/关闭, 否则会无法构建.

现在回到我们之前的项目 支持导入数据的 zookeeper <https://github.com/cnscud/cnscud-docker/tree/main/docker-zookeeper> , 之前构建的是使用了Intel芯片的Macbook, 
所以架构支持 amd64, 显然是不支持arm64的, 现在有了buildx, 可以同时支持多种架构了

## 运行构建:
```shell
docker buildx build --platform linux/amd64,linux/arm64 -t cnscud/zookeeper:zk3.6-0.2 . --push
```

然后我们去验证一下 <https://hub.docker.com/r/cnscud/zookeeper/tags>, 发现 3.6-0.2这个版本就支持2种架构了.

如图:

![多平台架构支持](/img/docker/zk-docker.arm64.png )


## 参考
* Buildx官方文档 <https://github.com/docker/buildx>
* 一篇中文博客, 很详细 <https://cloud.51cto.com/art/202108/678858.htm> 
