---
layout: post 
title: 修改Eureka的metadata脚本 
date:   2021-06-07 11:11 
description: 修改Eureka的metadata脚本 
categories: springcloud 
comments: true 
tags:
- eureka,springcloud

---
最近研究了一下Spring Cloud的灰度发布, 发现方法真是多. 这里先提供一个修改Eureka注册中心里的instance实例的metadata的脚本, 可以方便地用来测试效果.

使用举例: sh eureka.sh BETAZONE-HELLO-REMOTENAME 172.18.0.145 9001 zone main

* 你的应用服务名字为: BETAZONE-HELLO-REMOTENAME
* 实例服务的地址: 172.18.0.145
* 实例服务的端口: 9001
* 要修改的metadata key: zone
* 要修改的metadata 值: main


脚本如下, github地址: https://github.com/cnscud/javaroom/tree/main/betazone2

```shell
#!/bin/bash

###
### 修改eureka的instance的metadata.
###
### @author Felix Zhang 2021.6.7
###

# eureka host 服务器地址, 修改为自己的真实地址
eurekaHost='127.0.0.1:8001'


inputready=1

if [ ! -n "$1" ]; then
    echo "command error:serviceName empty!"
    inputready=0
fi

if [ ! -n "$2" ]; then
    echo "command error:serviceIp empty!"
    inputready=0
fi

if [ ! -n "$3" ]; then
    echo "command error:servicePort empty!"
    inputready=0
fi

if [ ! -n "$4" ]; then
    echo "command error:metadata-key empty!"
    inputready=0
fi

if [ ! -n "$5" ]; then
    echo "command error:metadata-value empty!"
    inputready=0
fi

if [ $inputready -eq 0 ]; then
    echo "Command Format: $0 serviceName serviceIp servicePort metadata-key metadata-value"
    echo "Example: $0 serviceName1 192.168.0.105 9001 zone beta"
    exit
fi

#转为小写
serviceName=$(echo "$1" | tr '[:upper:]' '[:lower:]')
hostName=$2
servicePort=$3
metakey=$4
metavalue=$5

# download eureka service list xml
echo 'start download eureka service list xml!'
curl -X GET http://${eurekaHost}/eureka/apps > eureka.xml
if [ $? -eq 0 ]; then
  echo "download eureka service success!"
else
  echo "download eureka service fail!"
fi

findinstance=0
myinstanceId="$hostName:$serviceName:$servicePort"

i=1
while true;
do
  xmllint --xpath "//instance[$i]/app/text()" eureka.xml >> /dev/null 2>&1
  if [ $? -eq 0 ]; then
    appName=$(xmllint --xpath "//instance[$i]/app/text()" eureka.xml)
    ipAddr=$(xmllint --xpath "//instance[$i]/ipAddr/text()" eureka.xml)
    port=$(xmllint --xpath "//instance[$i]/port/text()" eureka.xml)
    instanceId=$(xmllint --xpath "//instance[$i]/instanceId/text()" eureka.xml)
    echo "instance: $appName $ipAddr $port --> $instanceId"

    if [ "$instanceId" == "$myinstanceId" ]; then
      findinstance=1
    fi
  else
    break
  fi

  #递增
  let i+=1
done

if [ $findinstance -eq 0 ]; then
  echo "not find your instance: $myinstanceId"
  exit
else
  echo "find your instance: $myinstanceId"
fi



curl -X PUT "$eurekaHost/eureka/apps/$serviceName/$instanceId/metadata?$metakey=$metavalue"


if [ $? -eq 0 ]; then
  echo "update $serviceName $hostName $metakey:$metavalue success!"
else
  echo "update $serviceName $hostName $metakey:$metavalue fail!"
fi


```