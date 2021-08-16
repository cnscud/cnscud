---
layout: post
title: CircuitBreaker断路器Fallback如何获取异常
date:   2021-08-16 15:23
description: CircuitBreaker断路器Fallback如何获取异常
categories: spring
comments: true
tags:
- spring,gateway,fallback
---

## 内部 Fallback
在Spring Cloud 2020新版里, 可以使用新版的 CircuitBreaker 断路器, 这里使用内部fallback, 配置如下:

```yaml
server:
  port: 8900

spring:
  application:
    name: ms-gateway
  main:
    allow-bean-definition-overriding: true
  cloud:
    zookeeper:
      connect-string: ${ZK_HOSTS:127.0.0.1:2181}
      discovery:
        enabled: true
        preferIpAddress: true
    loadbalancer:
      ribbon:
        enabled: false
    gateway:
      discovery:
        locator:
          lowerCaseServiceId: true
          enabled: true
      routes:
        - id: default
          uri: lb://ms-fundmain-service
          predicates:
            - Path=/fundmain/**
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args:
                name: fetchIngredients
                fallbackUri: forward:/defaultfallback
            - name: Retry
              args:
                retries: 2
                series: SERVER_ERROR

```

fallback拦截了服务端异常, defaultfallback的实现如下:

```java
@RestController
public class DefaultFallback {

    private static final Logger logger = LoggerFactory.getLogger(GatewayErrorAttributes.class);

    @RequestMapping("/defaultfallback")
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public Map<String, Object> defaultfallback() {

        Map<String, Object> map = new HashMap<>();
        map.put("code", 5001);
        map.put("msg", "服务异常");
        return map;
    }

}
```

这样就可以实现一个最简单的fallback, 如果要获取服务的实际错误信息, 则改造如下:

```java
@RestController
public class DefaultFallback {

    private static final Logger logger = LoggerFactory.getLogger(GatewayErrorAttributes.class);
    
    @RequestMapping(value = "/defaultfallback")
    @ResponseStatus
    public Mono<Map<String, Object>> fallback(ServerWebExchange exchange) {
        Map<String, Object> result = new HashMap<>(4);
        result.put("code", 5001);

        Exception exception = exchange.getAttribute(ServerWebExchangeUtils.CIRCUITBREAKER_EXECUTION_EXCEPTION_ATTR);

        ServerWebExchange delegate = ((ServerWebExchangeDecorator) exchange).getDelegate();
        logger.error("服务调用失败，URL={}", delegate.getRequest().getURI(), exception);

        result.put("uri", delegate.getRequest().getURI());

        if (exception instanceof TimeoutException) {
            result.put("msg", "服务超时");
        }
        else if (exception != null && exception.getMessage() != null) {
            result.put("msg", "服务错误: " + exception.getMessage());
        }
        else {
            result.put("msg", "服务错误");
        }
        return Mono.just(result);
    }
}

```

这次注入了一个 ServerWebExchange, 然后获取保存在 CIRCUITBREAKER_EXECUTION_EXCEPTION_ATTR 里面的实例, 拿到相应的异常信息.


## 外部Fallback

    请参考 https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/, 使用 FallbackHeaders 即可.

```yaml

spring:
  application:
    name: ms-gateway
  main:
    allow-bean-definition-overriding: true
  redis:
    host: 127.0.0.1
    port: 6379
    database: 0
  cloud:
    zookeeper:
      connect-string: ${ZK_HOSTS:127.0.0.1:2181}
      discovery:
        enabled: true
        preferIpAddress: true
    loadbalancer:
      ribbon:
        enabled: false
    gateway:
      discovery:
        locator:
          lowerCaseServiceId: true
          enabled: true
      routes:
        - id: testoutsidefallback
          uri: lb://ms-fundmain-service
          predicates:
            - Path=/fundmain2/**
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args:
                name: fetchIngredients2
                fallbackUri: forward:/externalfallback
        - id: ingredients-fallback
          uri: lb://ms-fundmain-service
          predicates:
            - Path=/externalfallback
          filters:
            - name: FallbackHeaders
              args:
                executionExceptionTypeHeaderName: Exception-Type
                executionExceptionMessageHeaderName: Exception-Message
```

支持四个字段:
* executionExceptionTypeHeaderName ("Execution-Exception-Type")
* executionExceptionMessageHeaderName ("Execution-Exception-Message")
* rootCauseExceptionTypeHeaderName ("Root-Cause-Exception-Type")
* rootCauseExceptionMessageHeaderName ("Root-Cause-Exception-Message")

实现示例 Fallback如下: (**注意, 此Controller不能放在网关项目里, 否则就要用内部Fallback才行**)

```java
@RestController
public class ExternalFallback {

    private static final Logger logger = LoggerFactory.getLogger(ExternalFallback.class);

    private static final String EXECUTION_EXCEPTION_TYPE = "Exception-Type";
    private static final String EXECUTION_EXCEPTION_MESSAGE = "Exception-Message";

    @RequestMapping("/externalfallback")
    @ResponseStatus
    public Map<String, Object> externalfallback(HttpServletRequest request) {

        Map<String, Object> map = new HashMap<>();
        map.put("code", 5002);
        map.put("msg", "服务异常");

        map.put(EXECUTION_EXCEPTION_TYPE, request.getHeader(EXECUTION_EXCEPTION_TYPE));
        map.put(EXECUTION_EXCEPTION_MESSAGE, request.getHeader(EXECUTION_EXCEPTION_MESSAGE));

        return map;
    }
}

```



参考文章: 网上一篇关于Hystrix的文章
