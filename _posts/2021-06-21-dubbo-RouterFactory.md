---
layout: post
title: Dubbo 实现一个Route Factory(用于灰度发布)
date:   2021-06-21 11:57
description: 实现一个Route Factory
categories: dubbo
comments: true
tags:
- dubbo, route
---

Dubbo 可以实现的扩展很多, 官方文档在这: https://dubbo.apache.org/zh/docs/v2.7/dev/impls/  (太简单了....)

下面我们实现一个Route Factory, 它会根据参数中的workzone来选择合适的Invoker实例, 可以实现一定程度上的灰度发布.

首先实现一个Route, 
```java
package com.cnscud.dubboroom.base.dubbo;

import org.apache.dubbo.common.URL;
import org.apache.dubbo.rpc.Invocation;
import org.apache.dubbo.rpc.Invoker;
import org.apache.dubbo.rpc.RpcException;
import org.apache.dubbo.rpc.cluster.router.AbstractRouter;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.ArrayList;
import java.util.List;

/**
 * My Router.
 *
 * @author Felix Zhang 2021-06-21 10:38
 * @version 1.0.0
 */
public class MyRouter extends AbstractRouter {

    private static Logger logger = LoggerFactory.getLogger(MyRouter.class);
    protected static String ZONE_KEY = "workzone";

    @Override
    public <T> List<Invoker<T>> route(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException {

        //自己的逻辑
        String workzone = invocation.getAttachment("workzone");

        List<Invoker<T>> newInvokerList = new ArrayList<>();

        //选择特定服务器
        for (Invoker<T> invoker : invokers) {
            URL serviceUrl = invoker.getUrl();
            logger.info("router serviceUrl: " + serviceUrl.toIdentityString() + " " + serviceUrl.getParameters() + ", port: " + serviceUrl.getPort());

            if (serviceUrl.hasParameter(ZONE_KEY) && workzone != null && workzone.equalsIgnoreCase(serviceUrl.getParameter(ZONE_KEY))) {
                logger.info("find match invoker for workzone: " + workzone + " ip: " + serviceUrl.getIp() + " port: " + serviceUrl.getPort());
                newInvokerList.add(invoker);
            }
        }

        if (newInvokerList.size() > 0) {
            return newInvokerList;
        }

        return invokers;
    }

}

```

然后实现一个 Factory,
```java
package com.cnscud.dubboroom.base.dubbo;

import org.apache.dubbo.common.URL;
import org.apache.dubbo.rpc.cluster.CacheableRouterFactory;
import org.apache.dubbo.rpc.cluster.Router;

/**
 * My Router Factory.
 *
 * @author Felix Zhang 2021-06-21 10:48
 * @version 1.0.0
 */
public class MyRouterFactory extends CacheableRouterFactory {
    @Override
    protected Router createRouter(URL url) {
        return new MyRouter();
    }
}

```

在配置文件里声明:
声明 Route Factory: 文件名: META-INF/dubbo/org.apache.dubbo.rpc.cluster.RouterFactory
```ini
myrouter=com.cnscud.dubboroom.base.dubbo.MyRouterFactory
```

然后在dubbo的配置里声明使用:
```yaml
dubbo:
  consumer:
    router: myrouter
```

这样就可以生效了...... 

(仅供参考!!)
