---
layout: post
title: QT 如何在调试时能进入源码方式
date:   2020-04-24 15:01
description: 其实很简单, 你总是搞不对
categories: qt
comments: true
tags:
- 
---

最近在学习QT, 遇到一些crash, 也没看过QT源码啊, 就想类似Java一样, 在出错时进入源码跟踪一下, 但是QT和Java太不一样了, 死活进不去.

研究了几天, 发现本来是很简单的事情, 但是网上的文章让人容易钻进死胡同

## 说起来简单 
1. 用 Online Installer, 选中 "Qt Debug Information Files"
2. 安装的时候, 还要选中 Source
3. Qt Creator 自己的debug包不是QT的Debug包, 我们做自己的项目不需要, 除非你要开发Qt Creator自己的插件...
4. QT Creator 的Debugger中设置 Attache Qt Source 到 $安装目录/5.14.2/Src

![安装选择](/img/posts/qt_install_sourcedebug.jpg ) 

## 为啥有些人死活找不到
* 为了避免在线安装,很多教程都说网速慢... 用的不是Online Install, 里面没有 "Qt Debug Information Files" --其实国内有镜像, 速度都不错
* 可能Windows/Linux/MacOS三个系统的调试包安装不一样?
* Qt Creator 自己的debug包不是QT的Debug包, 容易理解错误
* 自己编译太慢了, 我编译了20个小时.... 还得生成Debug Symbols....太累了


## 如何使用镜像安装 MaintenanceTool
* 启动安装后, 设置用户存储库为镜像地址. 例如MacOS设置为: https://mirrors.tuna.tsinghua.edu.cn/qt/online/qtsdkrepository/mac_x64/root/qt/
* 可以随时增删组件, 方便啊, 增加Qt新版本啊

![设置镜像](/img/posts/onlineinstall_mirror.jpg "设置镜像")

## 自己编译
自己编译也是可以的, 而且MacOS我几乎啥也没准备, 就开始编译了....只遇到一个SDK小版本解析的error ,解决了. 就是编译用了了20多个小时.... (里面有个chromium ...)

## 示例
![示意](/img/posts/qtcreator_debug_source.jpg "能看QT的源码了, 不是头文件")
