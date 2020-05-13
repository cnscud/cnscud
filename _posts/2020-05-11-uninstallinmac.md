---
layout: post
title: Mac卸载软件真不省心啊
date:   2020-05-11 16:32
description: 
categories: macos
comments: true
tags: 
- macos,uninstall
 
---
最近看看磁盘觉得有点小, 就整理了一下, 经过一番折腾, 发现MacOS卸载软件可真是不省心啊. 从应用里移到垃圾桶仅仅是第一步, 当然对于不读写任何文件的应用也许就可以了.

咱们看看赶紧卸载一个软件, 需要影响那些目录. 
 
 以下仅为示例, 是我测试安装一个软件时候的, 结果留下一大堆垃圾, 要是处女座还不得哭死...

* /Library/Caches/ 缓存文件
* ~/Library/Caches/ 缓存文件
* /Library/Application Support/*** Tracker 365  文件存储目录
* ~/Library/Application Support/*** Tracker 365 文件存储目录
* /Library/Preferences/com.equinux.***Tracker365.plist 配置
*  ~/Library/Preferences/com.equinux.***Tracker365.plist 配置
* /Library/PrivilegedHelperTools/com.equinux.***Tracker365.connectiond 
* /Library/LaunchDaemons/com.equinux.***Tracker365.agent.plist 系统启动
* /Library/Extensions/com.equinux.***Tracker365.* 系统扩展(例如驱动, 网络等等)
* /System/Library/Extensions 系统扩展


到处都能看到各种CleanApp, CleanMac, MacBooster, 苹果怎么不出一个卸载工具哪....