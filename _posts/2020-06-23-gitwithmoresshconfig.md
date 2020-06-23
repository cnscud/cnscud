---
layout: post
title: 配置多个git用的ssh key
date:   2020-06-23 14:37
description: 
categories: git
comments: true
tags:
- 
---

参考 http://www.sail.name/2018/12/16/ssh-config-of-mac/

有一点注意 Host 的名字和 HostName改为一致. 因为从git仓库复制的地址是全程.

```bash
Host code.aliyun.com
  HostName code.aliyun.com
  User git
  PreferredAuthentications publickey
  IdentityFile ~/.ssh/id_rsa_aliyuncode
```
