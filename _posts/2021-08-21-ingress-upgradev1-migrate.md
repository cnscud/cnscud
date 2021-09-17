---
layout: post
title: NGINX Ingress控制器1.0.0迁移文档(翻译)
date:   2021-08-27 09:55
description: NGINX Ingress 控制器 升级到1.0迁移文档
categories: ingress
comments: true
tags:
- ingress
---

# Ingress 是什么
* Ingress 是对k8s集群中服务的外部访问进行管理的 API 对象，典型的访问方式是 HTTP。
* Ingress 可以提供负载均衡、SSL 终结和基于名称的虚拟托管。

最近发布了 1.0.0正式版, 有很多不兼容的改动.


## 名词说明:
    注解 annotation
    资源 resource
    对象 object
    控制器 controller

=============================================================

本文原始英文文档在此: <https://kubernetes.github.io/ingress-nginx/#faq-migration-to-apiversion-networkingk8siov1>
2021.8.27

# 欢迎

这是 NGINX Ingress 控制器 的文档.

它基于 [Kubernetes Ingress resource](http://kubernetes.io/docs/user-guide/ingress/) 构建, 使用 [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#understanding-configmaps-and-pods) 来保存 NGINX 的配置.

要了解更多关于使用 Ingress的内容, 点击这里 [k8s.io](http://kubernetes.io/docs/user-guide/ingress/) .

## 开始

如果想快速开始请阅读这里 [Deployment](https://kubernetes.github.io/ingress-nginx/deploy/) .


# 常见问题FAQ - 版本升级到  networking.k8s.io/v1

- 请阅读这里 [关于废弃ingress api 版本的官方说明](https://kubernetes.io/blog/2021/07/26/update-with-ingress-nginx/) 如果你在K8s v1.22之前版本的集群里使用了ingress对象, 然后你想升级到 K8s v1.22, 那这个文档对你有用.

- 请阅读这里 [关于IngressClass 对象的官方文档](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class)

## 什么是  ingressClass , 为什么现在它对 Ingress-NGINX 控制器 用户如此重要 ?

IngressClass是一种 Kubernetes 资源, 请阅读后面的描述.
这个很重要, 安装Ingress-NGINX 控制器, 以前的版本不需要ingressClass对象. 但是从 1.0.0版本之后, 就必须需要ingressclass这个对象了.

在集群里, 如果有多个Ingress-NGINX 控制器, 那么所有控制器的实例必须知道它们服务的是什么类型的 Ingress对象. 设置 ingress 对象的 ingressClass 字段就可以让控制器知道.

```
_$ k ingressClass 说明                                                           
KIND:     IngressClass                                                               
VERSION:  networking.k8s.io/v1                     

DESCRIPTION:    
     IngressClass 
     IngressClass 表示 Ingress的类, 在 Ingress Spec 里使用. 
     使用 `ingressclass.kubernetes.io/is-default-class` 注解可以用来设置 IngressClass 是否为缺省的class.
     当一个单独的  IngressClass 资源的此选项被设置为 true 时, 新的Ingress资源如果没有指定class字段, 则缺省使用这个IngressClass.

FIELDS:                                   
   apiVersion   <string>                                                             
     APIVersion 定义一个对象的版本. 会转换已知的值到最新的内部值, 不认识的值则可能会被拒绝. 
     更多信息: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources
                                                                                     
   kind <string>                                                                     
     Kind 是一个字符串值, 代表了这个对象的 REST 资源. 服务器会根据这个推断请求提交的服务终端(endpoint). 
     此值不可修改. 使用驼峰式语法. 
     更多信息: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds

   metadata     <Object>                           
     标准对象的元数据. 更多信息:                                                           
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata

   spec <Object>                                   
     Spec用来定义 IngressClass 预期到达的状态. 更多信息:                                        
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status`

```

## 引发了什么变化 ?

主要有两大重要变更.

(变更 #1) 使用 K8s 1.21版本或更早版本, 使用如下方式来声明使用 ingress 资源 ;
- apiVersion: extensions/v1beta1
- apiVersion: networking.k8s.io/v1beta1

  (你会得到一个废弃警告, 但是ingress资源还是会被创建.)

从 K8s 1.22 版本之后, 你只能设置 "apiVersion:" 这个字段的值为 "networking.k8s.io/v1". 原因是 [关于废弃ingress api 版本的官方说明](https://kubernetes.io/blog/2021/07/26/update-with-ingress-nginx/).

(变更 #2) 当你升级到 K8s v1.22版本, 如果你已经在使用Ingress-NGINX 控制器, 已经存在的ingress对象在几种情况下不能正常工作. 阅读此文档来检查是不是你使用的场景.

## ingressClassName 字段是什么 ?

ingressClassName 是ingress对象的spec下的字段.

```
% k ingress.spec.ingressClassName 说明
KIND:     Ingress
VERSION:  networking.k8s.io/v1

FIELD:    ingressClassName <string>

DESCRIPTION:
     IngressClassName 是IngressClass集群资源的名字. 
     这个相关的IngressClass定义了用哪个控制器来实现这个资源.
     这个字段用来代替废弃的 "kubernetes.io/ingress.class" 注解. 
     为了保持向后兼容, 如果也同时注解了这个字段, 那优先使用这个设置的值.
     如果注解和此字段的设置不一致, 控制器会发出一个警告信息.
     如果一个Ingress对象没有指定 ingressClassName字段, 则它会被忽略(也就是没有创建).
     如果系统中有一个IngressClass对象设置了缺省选项, 而且当前Ingress没有设置ingressClassName字段, 则此Ingress也会使用系统缺省的设置.
     相关更多信息, 请阅读 IngressClass 的文档.
```
注意: spec.ingressClassName 优先级比 注解(annotation) 高.



## 在我的集群里只有一个 Ingresss-NGINX controller 实例. 我应该怎么办 ?
- 如果你只有一个 Ingress-NGINX 控制器实例, 而且你想用 ingressclass, 你应该在你的 ingressClass 资源声明里面添加 "ingressclass.kubernetes.io/is-default-class" 注解, 这样你新的Ingress对象就会使用缺省的设置.

在这种情况下, 你需要让你的控制器感知Ingress对象. 如果你有几个 Ingress对象, 而且他们没有设置  [ingressClassName](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#ingress-v1-networking-k8s-io) 字段, 
而且也没有设置注解 (`kubernetes.io/ingress.class`) , 那么你应该用 [--watch-ingress-without-class=true](## 这个 '--watch-ingress-without-class' 是什么设置 ?) 参数来启动你的 ingress-controller .

你可以使用 `.controller.watchIngressWithoutClass: true` 来配置你的 helm chart 安装.

我们强烈建议你按下面的方法来创建你的 ingressClass (译注: 缺省的配置里是没有开启的):
```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  labels:
    app.kubernetes.io/component: controller
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx
```
并在你的 Ingress对象声明中添加 "spec.ingressClassName=nginx"


## 在我的集群里有很多 ingress 对象. 我应该怎么办 ?
- 如果你不关心 ingressClass , 或者你有很多没有配置ingressClass 的ingress对象, 你可以使用 `--watch-ingress-without-class=true` 来启动你的  ingress-controller .

## 这个 '--watch-ingress-without-class' 是什么设置 ?
- 这个设置会作为一个参数, 传递给 ingress-controller 调用, 就像下面这样:

```shell
...
...
args:
  - /nginx-ingress-controller
  - --publish-service=$(POD_NAMESPACE)/ingress-nginx-dev-v1-test-controller
  - --election-id=ingress-controller-leader
  - --controller-class=k8s.io/ingress-nginx
  - --configmap=$(POD_NAMESPACE)/ingress-nginx-dev-v1-test-controller
  - --validating-webhook=:8443
  - --validating-webhook-certificate=/usr/local/certificates/cert
  - --validating-webhook-key=/usr/local/certificates/key
  - --watch-ingress-without-class=true
...
...
```

## 在我的集群里有很多 controller 而且已经用了 annotation ?
没问题, 它会依然保持工作, 不过我们强烈建议你测试验证一下.

## 在我的集群里有很多 controller 已经在运行了, 需要我使用新的规范吗?
在这种情况下, 你需要创建多个 ingressClasses (参加样例一). 但是一定要注意此时 ingressClass 工作在一种特别的方式下: 你需要修改 IngressClass 的 .spec.controller 的值来让控制器指向相应的ingressClass. 让我们看一些例子, 假设你现在有2个 Ingress类:

- Ingress-Nginx-IngressClass-1 , 其设置 .spec.controller 为 "k8s.io/ingress-nginx1"
- Ingress-Nginx-IngressClass-2 , 其设置 .spec.controller 为 "k8s.io/ingress-nginx2"

当部署你的 ingress控制器时, 你必须修改按如下方式修改  `--controller-class` 字段:

- Ingress-Nginx-Controller-nginx1 使用 `k8s.io/ingress-nginx1`
- Ingress-Nginx-Controller-nginx2 使用 `k8s.io/ingress-nginx2`

然后, 当你使用 IngressClassName = `ingress-nginx2` 来创建一个Ingress对象是, 它会根据 `controller-class=k8s.io/ingress-nginx2` 这个配置来查找控制器, 同时 `Ingress-Nginx-Controller-nginx2` 会监测指向 `ingressClass="k8s.io/ingress-nginx2` 的对象, 它会为这个对象提供服务, 而 `Ingress-Nginx-Controller-nginx1` 会忽略这个对象.

要注意, 如果你的 `Ingress-Nginx-Controller-nginx2` 启动时带了参数 `--watch-ingress-without-class=true` , 它会为一下对象提供服务:
- 没有指定 ingress-class 属性的对象
- 使用注解 `--ingress-class` 来设置并且内容相同的对象
- 使用 `--controller-class` 来设定, 也就是和指定 .spec.controller 字段一样的对象


## 为什么 helm chart 里缺省禁用了 ingressClassResource  ?
- 如果这个字段被设置开启(enabled), 而且原来的集群里已经存在 ingress对象, 那原来已经存在的 ingress对象不在被关注(honored), 只有你创建的新的对象才会继承 ingressClass 的值.
