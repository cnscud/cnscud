---
layout: post
title: Maven项目思考&实战
date:   2020-08-19 20:15
description: 
categories: maven,java
comments: true
tags:
- maven
---

## Maven项目规范
* 同一项目中所有模块版本保持一致
* 子模块统一继承父模块的版本
* 统一在顶层模块Pom的节中定义所有子模块的依赖版本号，子模块中添加依赖时不要添加版本号
* 开发测试阶段使用SNAPSHOT
* 生产发布使用RELEASE或(无后缀)正式版
* 新版本迭代只修改父POM中的版本和子模块依赖的父POM版本

## 部署目标
* 正确
* 快速
* 简单
* 自动

## 善用工具
* 插件 versions-maven-plugin: 管理自身版本号, 依赖版本号
* 插件 maven-release-plugin: 自动升级版本, 提交代码, 打TAG

## 场景1: 独立项目(对外)的发布流程

* 确定/更新依赖模块版本(第三方)
* 发布服务器: 更新代码
* 准备发布: mvn -B -DskipTests=true   release:clean release:prepare
   * 本地准备: 修改版本号从snapshot到无后缀版本(或自定义)
   * 本地准备: GIT仓库的新TAG: bfeaturemod-1.0.8  
   * 本地编译
   * 本地准备: 更新版本号为下一个snapshot版本
   * 本地准备: 提交
* 正式发布: mvn -DskipTests=true   release:perform
* 发布最新的snapshot版本: mvn -DskipTests=true deploy


## 场景2: 内部项目永远snapshot 
内部项目就很灵活了, 这里介绍一种发布流程.

### 前提条件
* 所有项目版本号永远是snapshot, 而且一般不升版本号, 为1.0-snapshot
* 生产仓库和开发仓库物理隔离
* 如果只有一台部署机器, 则只使用mvn install, 不使用deploy
* 每次发布都是全量发布(如果代码没有修改, 部署脚本(自己编写)会比较后自动跳过)
* 配置好依赖关系后, 会自动先compile和install依赖 (自己编写的部署脚本)
* 父子模块可以分别compile和install (被依赖的话会自动编译安装)

备注: 如果多台部署机器, 则需要deploy, 则需要激活Maven仓库的profile (区分生产和开发)

### 内部项目部署—发布步骤
* 如果有依赖项目, 则先发布依赖项目(人工或者脚本根据配置)
* 更新代码, 检查代码是否有更新, 如果没有更新则不发布
* 编译发布 mvn clean install –DskipTests=true
* 打TAG提交到GIT
* 部署: 复制包到远程服务器, stop/start

### 内部项目部署—支持的方式
* Web (jetty)
* Service(Assembly) +使用wrapper包装
* Spring Boot + Wrapper
* Command 自定义
* 自己扩展

需要自己准备脚本(复制粘贴), 依赖配置等(一次性)

### 内部项目部署配置—示例
![](/img/posts/mvn_config.jpg "配置文件")

![](/img/posts/mvn_deploy.jpg "部署系统")

## 场景3: 更复杂的项目开发部署
可以组合使用versions-maven-plugin , maven-release-plugin 来自动发布, 但会比较繁琐.

![](/img/posts/mvn_complexdeploy.jpg "复杂项目部署")

过于复杂, 则不推荐使用了.