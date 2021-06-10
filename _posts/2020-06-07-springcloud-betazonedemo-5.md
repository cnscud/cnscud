---
layout: post 
title: Spring Cloud分区发布实践(5)--定制ServiceInstanceListSupplier
date:   2021-06-07 17:35 
description: Spring Cloud分区发布实践 
categories: springcloud 
comments: true 
tags:
- Spring, Spring Cloud, Zone

---
章节:
* [Spring Cloud分区发布实践(1) 准备环境](/springcloud/2021/06/07/springcloud-betazonedemo-1.html)
* [Spring Cloud分区发布实践(2) 微服务](/springcloud/2021/06/07/springcloud-betazonedemo-2.html)
* [Spring Cloud分区发布实践(3) 网关和负载均衡](/springcloud/2021/06/07/springcloud-betazonedemo-3.html)
* [Spring Cloud分区发布实践(4) FeignClient](/springcloud/2021/06/07/springcloud-betazonedemo-4.html)
* [Spring Cloud分区发布实践(5) 定制ServiceInstanceListSupplier](/springcloud/2021/06/07/springcloud-betazonedemo-5.html)
* [Spring Cloud分区发布实践(6) 根据Header选择实例区域](/springcloud/2021/06/07/springcloud-betazonedemo-6.html)


现在我们简单地来定制二个 ServiceInstanceListSupplier, 都是zone-preference的变种.

为了方便, 我重新调整了一下项目的结构, 把一些公用的类移动到hello-pubtool 模块, 这样网关项目和Feign项目就能复用一样的类了.

## A. main和beta互不相通, 绝对隔离 (资源相对充裕)
回到最开始的目的,  我们先实现这个A方案

```java
package com.cnscud.betazone.pub.samezone;

import com.cnscud.betazone.pub.LogUtils;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.loadbalancer.config.LoadBalancerZoneConfig;
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;
import reactor.core.publisher.Flux;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

/**
 * 只返回统一区域的实例. (和网上的代码略有不同)
 * @see org.springframework.cloud.loadbalancer.core.ZonePreferenceServiceInstanceListSupplier
 * @see org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplierBuilder
 *
 */
public class SameZoneOnlyServiceInstanceListSupplier implements ServiceInstanceListSupplier {

    private final String ZONE = "zone";

    private final ServiceInstanceListSupplier delegate;

    private final LoadBalancerZoneConfig zoneConfig;

    private String zone;

    public SameZoneOnlyServiceInstanceListSupplier(ServiceInstanceListSupplier delegate,
                                                   LoadBalancerZoneConfig zoneConfig) {
        this.delegate = delegate;
        this.zoneConfig = zoneConfig;
    }

    @Override
    public String getServiceId() {
        return delegate.getServiceId();
    }

    @Override
    public Flux<List<ServiceInstance>> get() {
        return delegate.get().map(this::filteredByZone);
    }

    private List<ServiceInstance> filteredByZone(List<ServiceInstance> serviceInstances) {
        if (zone == null) {
            zone = zoneConfig.getZone();
        }

        if (zone != null) {
            List<ServiceInstance> filteredInstances = new ArrayList<>();
            for (ServiceInstance serviceInstance : serviceInstances) {
                String instanceZone = getZone(serviceInstance);
                if (zone.equalsIgnoreCase(instanceZone)) {
                    filteredInstances.add(serviceInstance);
                }
            }
            //如果没找到就返回空列表，绝不返回其他集群的实例
            LogUtils.warn("find instances size: " + filteredInstances.size());
            return filteredInstances;
        }

        //如果没有zone设置, 则返回所有实例
        return serviceInstances;
    }

    private String getZone(ServiceInstance serviceInstance) {
        Map<String, String> metadata = serviceInstance.getMetadata();
        if (metadata != null) {
            return metadata.get(ZONE);
        }
        return null;
    }

}

```

很简单, 不过要注意这个实现如果没有zone设置, 则返回所有可用实例.

对应的配置声明:
```java


/**
 * 自定义 Instance List Supplier: 根据默认Zone划分, 并且zone互相隔离.
 */
public class SameZoneOnlyCustomLoadBalancerConfiguration {

    @Bean
    public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(
            DiscoveryClient discoveryClient, Environment env,
            LoadBalancerZoneConfig zoneConfig,
            ApplicationContext context) {

        ServiceInstanceListSupplier delegate = new SameZoneOnlyServiceInstanceListSupplier(
                new DiscoveryClientServiceInstanceListSupplier(discoveryClient, env),
                zoneConfig
        );
        ObjectProvider<LoadBalancerCacheManager> cacheManagerProvider = context
                .getBeanProvider(LoadBalancerCacheManager.class);
        if (cacheManagerProvider.getIfAvailable() != null) {
            return new CachingServiceInstanceListSupplier(
                    delegate,
                    cacheManagerProvider.getIfAvailable()
            );
        }
        return delegate;

    }
    
}


```

这里使用了缓存, 如果需要更多特性, 就去 LoadBalancerClientConfiguration 的源码里去参悟吧, 还有什么 health-check, same-instance-preference很多东西可以参考.

## C. beta绝对隔离, main在发布过程中可以切换到beta (main区域只有一台机器, 资源比较紧张)

这个也比较简单, 仅有几行代码的差异

```java
package com.cnscud.betazone.pub.samezone;

import com.cnscud.betazone.pub.LogUtils;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.loadbalancer.config.LoadBalancerZoneConfig;
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;
import reactor.core.publisher.Flux;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

/**
 * Beta只返回统一区域的实例, 其他区域如果为空则返回所有实例.
 * @see org.springframework.cloud.loadbalancer.core.ZonePreferenceServiceInstanceListSupplier
 * @see org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplierBuilder
 *
 */
public class SameZoneSpecialBetaServiceInstanceListSupplier implements ServiceInstanceListSupplier {

    private final String ZONE = "zone";
    private String ZONE_BETA = "beta";

    private final ServiceInstanceListSupplier delegate;

    private final LoadBalancerZoneConfig zoneConfig;

    private String zone;

    public SameZoneSpecialBetaServiceInstanceListSupplier(ServiceInstanceListSupplier delegate,
                                                          LoadBalancerZoneConfig zoneConfig) {
        this.delegate = delegate;
        this.zoneConfig = zoneConfig;
    }

    @Override
    public String getServiceId() {
        return delegate.getServiceId();
    }

    @Override
    public Flux<List<ServiceInstance>> get() {
        return delegate.get().map(this::filteredByZone);
    }

    private List<ServiceInstance> filteredByZone(List<ServiceInstance> serviceInstances) {
        if (zone == null) {
            zone = zoneConfig.getZone();
        }

        if (zone != null) {
            List<ServiceInstance> filteredInstances = new ArrayList<>();
            for (ServiceInstance serviceInstance : serviceInstances) {
                String instanceZone = getZone(serviceInstance);
                if (zone.equalsIgnoreCase(instanceZone)) {
                    filteredInstances.add(serviceInstance);
                }
            }
            //如果没找到就返回空列表，绝不返回其他集群的实例
            LogUtils.warn("find instances size: " + filteredInstances.size());
            if(filteredInstances.size()>0) {
                return filteredInstances;
            }
            else {
                //如果是beta, 则返回空
                if (zone.equalsIgnoreCase(ZONE_BETA)){
                    return filteredInstances;
                }
            }
        }

        //如果没有zone设置, 则返回所有实例
        return serviceInstances;
    }

    private String getZone(ServiceInstance serviceInstance) {
        Map<String, String> metadata = serviceInstance.getMetadata();
        if (metadata != null) {
            return metadata.get(ZONE);
        }
        return null;
    }

}


```

这个也没啥特殊的.

## D. 发布前标注一套区域为beta的服务, 测试通过后修改beta服务的区域为main, 多个实例都可上线服务 (资源非常紧张, 操作相对复杂)

这个就是个附加操作, 可以使用A,B,C任意一个方案, 搭配脚本就可以使用.

脚本已经准备好了:
* [修改Eureka的metadata脚本](/springcloud/2021/06/07/eureka-update-metadata.html)


## 比较
灰度发布肯定还有很多方案, 但是对于作者来说, 根据zone来做分区灰度发布, 可能这是最简单的一种方式了, 实现简单, 通过Nginx做简单的设置分流到2组网关上, 就可以实现2组实例了.

当然Nginx也可以根据header, cookie, URL来做分流,就看自己的需要了.

代码简单, 容易理解, 大道至简!

后面还会实践通过Header来分流的方法, 不过比较而言, zone-preference 还是最简单的, 后面的实践起来服务直接传递数据就是头疼...
