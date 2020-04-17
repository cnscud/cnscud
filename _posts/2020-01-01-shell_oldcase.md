---
layout: post
title: 一些Shell脚本记录
date:   2020-04-17 15:26
description: 
comments: true
categories: linux
tags:
- shell,linux
---

## 查看换行符
> 使用vim打开文件，输入:set ff?。根据返回结果可以文件类型

## 字符串
> opcenter.sjb.bz

```bash
 [root@opcenter backup]# echo ${HOSTNAME%.*}  
 opcenter.sjb  
 [root@opcenter backup]# echo ${HOSTNAME##*.}  
 bz  
```

## 看配置文件
```bash
cat server.conf | grep "^[^#|^;]"
``` 

## 替换换行符
```bash
 cat ip.test |sed ':a N;s/\n/, /;ta'
 cat ip.test |awk '{printf("%s,",$1)}'
 cat ip.test |grep -E -v  '127.0.0.1|126.23.23.44'
```

 

## 分析Access log
```bash
 #查找access log中请求时间超过1秒的请求  
 cat ms.access.log |awk  '{s=NF-2; print $4, $7 , $s}' |awk '($3 > 1 ) {print $1, $2,$3}'  

 #查找下载量比较大的请求  
 cat *.access.log |grep "2015:01:0"|awk  '{s=NF-2; print $2,$7 , $10,$s}' |awk '($3 > 1000000 ) {print $1, $2,$3,$4}'   
```

 

## AWK比较两个字段不同
```bash
 awk ' $1 != $2 {print $1, $2}' weixin.1.log |grep "1502" 
```
#
## AWK切割, 分析网络请求
```bash
 awk '{print $5}' netstat050604.txt  |awk -F":" '{print $1}' |sort -nr |uniq -c |sort -nr |more
```



  
 

## AWK分析日志
```bash
# 美国:  
cat www.access.log |grep "ustest=forus "|grep -v "114.242.69.13" |grep -v "ApacheBench" |awk '{s=NF-2;t=NF-1; print $1,$7,$s,$t}'

# 北京: 
cat m.access.log.20150716 |grep "ustest=forus "|grep -v "114.242.69.13" |grep -v "ApacheBench" |awk '{s=NF-5;t=NF-6; print $1,$7,$s,$t}'   
```

 