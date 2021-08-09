---
layout: post
title: Spring Cloud Gateway自定义异常处理Exception Handler
date:   2021-08-09 20:02
description: Spring Cloud Gateway自定义异常处理Exception Handler
categories: springcloud
comments: true
tags:
- springcloud
---

版本: Spring Cloud 2020.0.3

常见的方法有 实现自己的 DefaultErrorWebExceptionHandler 或 仅实现ErrorAttributes.

## 方法1: ErrorWebExceptionHandler (仅供示意)

自定义一个 GlobalErrorAttributes:
```java

@Component
public class GlobalErrorAttributes extends DefaultErrorAttributes{

    @Override
    public Map<String, Object> getErrorAttributes(ServerRequest request, ErrorAttributeOptions options) {
        Throwable error = super.getError(request);

        Map<String, Object> map = super.getErrorAttributes(request, options);
        map.put("status", HttpStatus.BAD_REQUEST.value());
        map.put("message", error.getMessage());
        return map;
    }
}
```

实现一个 
```java

@Component
@Order(-2)
public class GlobalErrorWebExceptionHandler extends AbstractErrorWebExceptionHandler {

    public GlobalErrorWebExceptionHandler(GlobalErrorAttributes gea, ApplicationContext applicationContext,
                                          ServerCodecConfigurer serverCodecConfigurer) {
        super(gea, new WebProperties.Resources(), applicationContext);
        super.setMessageWriters(serverCodecConfigurer.getWriters());
        super.setMessageReaders(serverCodecConfigurer.getReaders());
    }

    //渲染html或json
    @Override
    protected RouterFunction<ServerResponse> getRoutingFunction(final ErrorAttributes errorAttributes) {
        return RouterFunctions.route(RequestPredicates.all(), this::renderErrorResponse);
    }

    private Mono<ServerResponse> renderErrorResponse(final ServerRequest request) {

        final Map<String, Object> errorPropertiesMap = getErrorAttributes(request, ErrorAttributeOptions.defaults());

        return ServerResponse.status(HttpStatus.BAD_REQUEST)
                .contentType(MediaType.APPLICATION_JSON)
                .body(BodyInserters.fromValue(errorPropertiesMap));
    }
}

```

## 方法2, 仅实现一个 ErrorAttributes, 以覆盖默认的 DefaultErrorAttributes

```java

//Spring 默认的就很好了.
@Component
public class GatewayErrorAttributes extends DefaultErrorAttributes {

    private static final Logger logger = LoggerFactory.getLogger(GatewayErrorAttributes.class);

    @Override
    public Map<String, Object> getErrorAttributes(ServerRequest request,  ErrorAttributeOptions options) {
        Throwable error = super.getError(request);
        Map<String, Object> errorAttributes = new HashMap<>(8);
        errorAttributes.put("message", error.getMessage());
        errorAttributes.put("method", request.methodName());
        errorAttributes.put("path", request.path());

        MergedAnnotation<ResponseStatus> responseStatusAnnotation = MergedAnnotations
                .from(error.getClass(), MergedAnnotations.SearchStrategy.TYPE_HIERARCHY).get(ResponseStatus.class);

        HttpStatus errorStatus = determineHttpStatus(error, responseStatusAnnotation);

        //必须设置, 否则会报错, 因为 DefaultErrorWebExceptionHandler 的 renderErrorResponse 方法会获取此属性, 重新实现 DefaultErrorWebExceptionHandler也可.
        errorAttributes.put("status", errorStatus.value());
        errorAttributes.put("code", errorStatus.value());

        //html view用
        errorAttributes.put("timestamp", new Date());
        //html view 用
        errorAttributes.put("requestId", request.exchange().getRequest().getId());

        errorAttributes.put("error", errorStatus.getReasonPhrase());
        errorAttributes.put("exception", error.getClass().getName());

        return errorAttributes;
    }

    //从DefaultErrorWebExceptionHandler中复制过来的
    private HttpStatus determineHttpStatus(Throwable error, MergedAnnotation<ResponseStatus> responseStatusAnnotation) {
        if (error instanceof ResponseStatusException) {
            return ((ResponseStatusException) error).getStatus();
        }
        return responseStatusAnnotation.getValue("code", HttpStatus.class).orElse(HttpStatus.INTERNAL_SERVER_ERROR);
    }
    
}
```

这样就可以了.  

**注意注意**:   必须设置  errorAttributes.put("status", errorStatus.value()) , 否则会报错, 因为 DefaultErrorWebExceptionHandler 的 renderErrorResponse 方法会获取此属性. 除非你自己像方法一一样重新实现 DefaultErrorWebExceptionHandler.

然后在网关中访问一个不存在的服务, 即可看到效果.

```shell
curl 'http://127.0.0.1:8900/fundmain22/abc/gogogo?id=1000' --header 'Accept: application/json'
{"exception":"org.springframework.web.server.ResponseStatusException","path":"/fundmain22/abc/gogogo","code":404,"method":"GET","requestId":"094e53e5-1","message":"404 NOT_FOUND","error":"Not Found","status":404,"timestamp":"2021-08-09T11:07:44.106+0000"} 
```

