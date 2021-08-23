---
layout: post
title: 支持初始化数据的Zookeeper Docker镜像
date:   2021-08-23 15:06
description: 第一次启动可以从文件导入初始化数据
categories: docker
comments: true
tags:
- docker,zookeeper
---

最近在做一个演示项目 <https://github.com/cnscud/cavedemo>, 自然为了方便, 也做了docker打包, 发现zookeeper的镜像没有导入初始化数据的功能, 于是自己做了一个镜像, 还是蛮简单的.

镜像已经发布在 <https://hub.docker.com/r/cnscud/zookeeper> 官方仓库.

## 使用方法1: 命令行直接运行 
没参数的话和官方镜像一样, 也可以指定参数, 建议用docker-compose.

```shell
docker run --name zk1 -d -p 2181:2181 cnscud/zookeeper:zk3.6-0.1
```

## 使用docker-compose, 方便设定参数
下面是一个例子: `samples/docker-compose1/docker-compose.yml`:

```yaml
version: '3'
services:
  cnscud-zookeeper:
    image: cnscud/zookeeper:zk3.6-0.1
    volumes:
      - ./init.data:/init.data/
      - ./rundata/data:/data
      - ./rundata/logs:/datalog
    container_name: cnscud-sample-zk1
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"
    restart: on-failure
    ## use the hostname in project
    hostname: cnscud-sample-zk1.cnscud.com
```

你可以在  samples/docker-compose1 目录下发现所需的所有文件: https://github.com/cnscud/cnscud-docker/tree/main/docker-zookeeper/samples/docker-compose1

## 初始化数据使用说明
我们可以看到, 在volume里绑定了一个 /init.data/ 目录, 那么还有什么特定规则哪?

需要准备一个 脚本文件 "import.data.zk.sh", 放在这个目录里, 用来初始化数据, 至于文件内容, 就根据你的需要来编写了.

下面是一个例子, 是从文件里读取节点内容, 并设置到zookeeper的节点上去.


```shell
#!/bin/bash

## for init zookeeper data, you need update this file.
##
## author: felix zhang https://github.com/cnscud/  2021.8.22
##
## please make sure the file 755
##


CMD=`which zkCli.sh`
find="1"
if [ -z $CMD ]
then
	find="0"
fi

if [ $find = "0" ]
then
	CMD="$ZK_HOME/bin/zkCli.sh"
fi

echo $CMD

if [ -z $CMD ]
then
  echo "not found zkCli.sh, please check!!!"
  exit 1
fi


$CMD  create /xpower "1"
$CMD  create /xpower/cache "1"
$CMD  create /xpower/config "1"
$CMD  create /xpower/dbn "1"

## read from file!!!
## read from file!!!
## read from file!!!
$CMD  create /xpower/cache/redis.test "`cat /init.data/redis.test.conf`"
$CMD  create /xpower/config/kafka "`cat /init.data/kafka.conf`"
$CMD  create /xpower/dbn/cavedemo "`cat /init.data/mysql.cavedemo.conf`"

```

我们可以看到这个脚本做了几件事:
* 查找zkCli.sh, 并设置正确的路径
* 创建各级节点
* 创建最终的节点, 节点的内容是从文件读取的, 可以支持换行和双引号等, 要特别注意这个语法.

在这个脚本里, 使用zookeeper自带的zkCli.sh 脚本创建你的节点, 可以通过下面的命令**从文件读入节点内容**:
> "`cat /init.data/kafka.conf`"

要注意这个脚本里用到的文件要实现放到统一目录下, 或者其他绑定的目录下.

## 更多配置选项
    因为继承的是官方的zookeeper docker镜像, 所以更多信息可以参考 https://hub.docker.com/_/zookeeper.

## 原理分析
我们用 docker inspect 或者直接查看git仓库可以看到官方docker的设置:

```dockerfile
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["zkServer.sh", "start-foreground"]
```

如果我们需要导入数据, 则需要先后台启动zookeeper, 执行导入操作, 然后重启zookeeper, 不过要放在foreground方式启动.
看看这个重写的dockerfile:
```dockerfile
FROM zookeeper:3.6

##
## docker image for zookeeper with init data feature.
##
## author: Felix Zhang<cnscud@gmail.com>  2021.8.23
## website: https://github.com/cnscud/cnscud-docker

LABEL OG=cnscud.com
LABEL maintainer="Felix Zhang<cnscud@gmail.com>"
LABEL reference="https://github.com/cnscud/cnscud-docker"

## my entrypoint script
ADD cnscud-docker-entrypoint.sh /

## timezone
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

## declare the volumen for init data
VOLUME /init.data

## declare the port
EXPOSE 2181

## ===================================

## from official zookeeper as reference
# ENTRYPOINT ["/docker-entrypoint.sh"]
# CMD ["zkServer.sh" "start-foreground"]

ENTRYPOINT ["/cnscud-docker-entrypoint.sh"]
CMD ["zkServer.sh","start-foreground"]

```

里面重新声明了ENTRYPOINT, 换成了自定义版本的 cnscud-docker-entrypoint.sh.  里面还声明了VOLUME  /init.data . 

当然我们尽量复用原有的脚本, 而不是改的面目全非(毕竟那么乱...), 我们来看看:

```shell
#!/bin/bash
## script for support import data with zookeeper
## author: felix zhang 2021.8.22
## please make sure the file 755

# set -e

initedfile="/data/zk.cnscud.inited"
initcmdfile='/init.data/import.data.zk.sh'

needcallinit="Y"

#
if [ ! -f "$initedfile" ]
then
  needcallinit="Y${needcallinit}"
else
  echo "[cnscud] data had inited, will not init again."
fi

if [ -f "$initcmdfile" ]
then
  needcallinit="Y${needcallinit}"
else
  echo "[cnscud] not found script for init data, skip."
fi

echo "check needcallinit is: $needcallinit"

## start my import data ====================
if [ $needcallinit = "YYY" ]
then

  ## start in backgroup by original entry point
  /docker-entrypoint.sh zkServer.sh start

  ## waiting for zookeeper
  sleep 10

  echo "[cnscud] call init data now..."
  date > "$initedfile"

  ##import data
  sh $initcmdfile >> $initedfile

  /docker-entrypoint.sh zkServer.sh stop

  ## mark
  echo "[cnscud] mark: init was finished!"
  date >> "$initedfile"
  echo "done" >> "$initedfile"
fi

## end my import data ====================


echo "[cnscud] Starting Zookeeper in foreground mode..."
/docker-entrypoint.sh $@
```

我们可以看到这个代码其实很简单, 虽然很多行:
* 首选判断是否已经执行过, 用 /data/zk.cnscud.inited 文件方式来标记
* 如果没有初始化过, 则后台方式启动zkServer
* 执行初始化脚本
* 停止 zkServer
* 重新按默认参数或传入的参数启动zkServer.


## 感想
其实很简单, 只不过第一次发布镜像到docker hub, 没想到社区非常开放, 任何人都能发布镜像上去, 就导致很混乱, 很多镜像没有说明, 没有人维护, 但下载量很大. 希望不要像npm那么出现问题.

docker很好, 准备好内存和硬盘~!

## 感谢
感谢google, stackoverflow, withpy 以及我的家人.
