# hy-common-app 架构设计文档

## 1. 模块概述

### 1.1 模块定位

- **模块名称**：hy-common-app
- **父模块**：hy-common
- **模块类型**：后端公共应用模块
- **核心职责**：为所有后端 app 模块提供统一的 AOP 切面、拦截器、公共 Entity 等基础设施能力

### 1.2 设计目标

- 统一请求拦截：通过过滤器/拦截器实现请求统一处理
- 分层日志记录：基于 AOP 实现 Controller/Manager/Service 三层统一日志记录
- 接口幂等处理：基于 requestId + 重试标志实现幂等控制
- 权限验证集成：支持功能权限和数据权限验证
- 公共 Entity 封装：提供统一的实体基类，包含审计字段
- 分表支持：提供分表拦截器和分表实体基类

### 1.3 功能范围

| 功能 | 说明 | 状态 |
| ---- | ---- | ---- |
| 统一拦截器 | 拦截所有请求，验证 traceId | 必须 |
| Controller AOP | 日志记录、幂等处理、权限验证、审计日志 | 必须 |
| Manager AOP | 事务管理、日志记录 | 必须 |
| Service AOP | 日志记录、实体字段自动赋值、流水表记录 | 必须 |
| Mapper 拦截器 | 分表、数据权限、分页、SQL日志 | 必须 |
| 公共 Entity | 包含审计字段的实体基类 | 必须 |
| 公共分表 Entity | 继承公共 Entity，支持分表字段 | 必须 |

## 2. 架构设计

### 2.1 整体架构图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           请求入口层                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    GlobalFilter / Interceptor                        │   │
│  │  ┌──────────────┐  ┌──────────────┐                              │   │
│  │  │ TraceFilter  │→│ RequestFilter│                              │   │
│  │  │ 验证traceId  │  │ 请求预处理   │                              │   │
│  │  │ 无则返回403  │  │ 包装request  │                              │   │
│  │  └──────────────┘  └──────────────┘                              │   │
│  └────────────────────────────────┬───────────────────────────────────┘   │
└───────────────────────────────────┼───────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           Controller 层                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                     Controller AOP (Aspect)                          │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │  LogAspect   │  │IdempotentAspect││ AuthAspect   │              │   │
│  │  │  日志记录    │  │  幂等处理     │  │  权限验证    │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  │                     ┌──────────────┐                                 │   │
│  │                     │ AuditAspect  │                                 │   │
│  │                     │  审计日志    │                                 │   │
│  │                     └──────────────┘                                 │   │
│  └────────────────────────────────┬───────────────────────────────────┘   │
└───────────────────────────────────┼───────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           Manager 层                                      │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                     Manager AOP (Aspect)                             │   │
│  │  ┌──────────────┐  ┌──────────────┐                                 │   │
│  │  │  LogAspect   │  │TransactionAspect│                             │   │
│  │  │  日志记录    │  │  事务管理    │                                 │   │
│  │  └──────────────┘  └──────────────┘                                 │   │
│  └────────────────────────────────┬───────────────────────────────────┘   │
└───────────────────────────────────┼───────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           Service 层                                      │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                     Service AOP (Aspect)                             │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │  LogAspect   │  │ EntityAspect │  │ JournalAspect│              │   │
│  │  │  日志记录    │  │ 字段自动赋值 │  │  流水表记录  │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  └────────────────────────────────┬───────────────────────────────────┘   │
└───────────────────────────────────┼───────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           Mapper 层                                       │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    MyBatis Interceptor Chain                        │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │ TableShardInterceptor ││ DataPermissionInterceptor││PageInterceptor││   │
│  │  │    分表处理   │  │   数据权限    │  │   分页限制   │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  │                     ┌──────────────┐                                 │   │
│  │                     │ SqlLogInterceptor│                             │   │
│  │                     │   SQL日志记录   │                             │   │
│  │                     └──────────────┘                                 │   │
│  └────────────────────────────────┬───────────────────────────────────┘   │
└───────────────────────────────────┼───────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           数据库层                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│  │ 主表     │  │ 分表A    │  │ 分表B    │  │ 流水表   │                 │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘                 │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件分层

| 层级 | 组件名称 | 职责说明 |
| ---- | -------- | -------- |
| 拦截器层 | TraceFilter | 验证请求头中的 traceId，不存在则返回403 |
| 拦截器层 | RequestFilter | 请求预处理，包装request支持多次读取body |
| AOP层 | ControllerLogAspect | Controller 日志记录（warn/error级别） |
| AOP层 | IdempotentAspect | 接口幂等处理（三态管理） |
| AOP层 | AuthAspect | 权限验证（功能权限、数据权限） |
| AOP层 | AuditAspect | 审计日志记录（成功/失败均记录） |
| AOP层 | ManagerLogAspect | Manager 日志记录（info/error级别） |
| AOP层 | ServiceLogAspect | Service 日志记录（debug/error级别，含出参） |
| AOP层 | EntityFillAspect | Entity 字段自动赋值（含wrapper方式） |
| AOP层 | JournalAspect | 流水表记录（含删除操作） |
| 拦截器层 | TableShardInterceptor | MyBatis 分表拦截器 |
| 拦截器层 | DataPermissionInterceptor | 数据权限拦截器 |
| 拦截器层 | PageInterceptor | 分页大小限制拦截器（操作IPage对象） |
| 集成组件 | p6spy | SQL日志记录（JDBC代理） |
| 实体层 | BaseEntity | 公共实体基类（实现Serializable） |
| 实体层 | ShardEntity | 分表实体基类 |
| 上下文层 | RequestContextHolder | 线程上下文管理器 |

## 3. 模块结构

### 3.1 目录结构

```
hy-common-app/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/houyu/common/app/
│   │   │       ├── config/
│   │   │       │   ├── AppAutoConfiguration.java
│   │   │       │   ├── AppProperties.java
│   │   │       │   └── MyBatisPlusConfig.java
│   │   │       ├── interceptor/
│   │   │       │   ├── TraceFilter.java
│   │   │       │   ├── RequestFilter.java
│   │   │       │   └── WebMvcConfig.java
│   │   │       ├── aop/
│   │   │       │   ├── controller/
│   │   │       │   │   ├── ControllerLogAspect.java
│   │   │       │   │   ├── IdempotentAspect.java
│   │   │       │   │   ├── AuthAspect.java
│   │   │       │   │   └── AuditAspect.java
│   │   │       │   ├── manager/
│   │   │       │   │   └── ManagerLogAspect.java
│   │   │       │   └── service/
│   │   │       │       ├── ServiceLogAspect.java
│   │   │       │       ├── EntityFillAspect.java
│   │   │       │       └── JournalAspect.java
│   │   │       ├── mybatis/
│   │   │       │   ├── TableShardInterceptor.java
│   │   │       │   ├── DataPermissionInterceptor.java
│   │   │       │   └── PageInterceptor.java
│   │   │       ├── entity/
│   │   │       │   ├── BaseEntity.java
│   │   │       │   └── ShardEntity.java
│   │   │       ├── context/
│   │   │       │   └── RequestContextHolder.java
│   │   │       ├── annotation/
│   │   │       │   ├── RequirePermission.java
│   │   │       │   ├── AuditLog.java
│   │   │       │   └── Idempotent.java
│   │   │       ├── enums/
│   │   │       │   ├── OpType.java
│   │   │       │   └── IdempotentStatus.java
│   │   │       ├── service/
│   │   │       │   ├── IdempotentService.java
│   │   │       │   ├── PermissionService.java
│   │   │       │   └── JournalService.java
│   │   │       └── util/
│   │   │           └── SqlUtils.java
│   │   └── resources/
│   │       ├── META-INF/
│   │       │   └── spring/
│   │       │       └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
│   │       └── spy.properties
│   └── test/
│       └── java/
│           └── com/houyu/common/app/
│               ├── aop/
│               │   └── ControllerLogAspectTest.java
│               └── mybatis/
│                   └── TableShardInterceptorTest.java
└── pom.xml
```

### 3.2 关键类职责

| 类名 | 职责 | 所属包 |
| ---- | ---- | ------ |
| TraceFilter | 验证 X-Trace-Id，不存在返回403 | interceptor |
| RequestFilter | 请求预处理，包装ContentCachingRequestWrapper | interceptor |
| ControllerLogAspect | 记录 Controller 层日志，含requestBody/responseHeaders | aop.controller |
| IdempotentAspect | 幂等处理，三态管理（处理中/成功/失败） | aop.controller |
| AuthAspect | 权限验证，支持注解和全局配置 | aop.controller |
| AuditAspect | 审计日志记录，成功(INFO)/失败(ERROR)均记录 | aop.controller |
| ManagerLogAspect | 记录 Manager 层日志，含完整方法签名 | aop.manager |
| ServiceLogAspect | 记录 Service 层日志，含出参 | aop.service |
| EntityFillAspect | 自动填充 entity 审计字段（含wrapper方式） | aop.service |
| JournalAspect | 发送消息到流水表（含删除操作） | aop.service |
| TableShardInterceptor | MyBatis 分表拦截器 | mybatis |
| DataPermissionInterceptor | 数据权限 SQL 改写 | mybatis |
| PageInterceptor | 分页大小限制（操作IPage对象，最大5000） | mybatis |
| BaseEntity | 公共实体基类，实现Serializable，含自动填充注解 | entity |
| ShardEntity | 分表实体基类 | entity |
| RequestContextHolder | 线程上下文管理器 | context |
| IdempotentService | 幂等缓存管理，三态管理 | service |

## 4. 核心功能设计

### 4.1 统一拦截器

#### 4.1.1 TraceFilter 实现

```java
package com.houyu.common.app.interceptor;

import com.houyu.common.app.context.RequestContextHolder;
import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class TraceFilter implements Filter {

    private static final String TRACE_ID_HEADER = "X-Trace-Id";

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        
        String traceId = httpRequest.getHeader(TRACE_ID_HEADER);
        if (traceId == null || traceId.isEmpty()) {
            httpResponse.setStatus(HttpServletResponse.SC_FORBIDDEN);
            httpResponse.getWriter().write("X-Trace-Id is required");
            return;
        }
        
        RequestContextHolder.setTraceId(traceId);
        
        try {
            chain.doFilter(request, response);
        } finally {
            RequestContextHolder.clear();
        }
    }
}
```

#### 4.1.2 RequestFilter 实现

```java
package com.houyu.common.app.interceptor;

import com.houyu.common.app.context.RequestContextHolder;
import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.web.util.ContentCachingRequestWrapper;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 1)
public class RequestFilter implements Filter {

    private static final String REQUEST_ID_HEADER = "X-Request-Id";
    private static final String USER_ID_HEADER = "X-User-Id";
    private static final String USER_NAME_HEADER = "X-User-Name";
    private static final String RETRY_FLAG_HEADER = "X-Retry-Flag";

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        
        ContentCachingRequestWrapper wrappedRequest = new ContentCachingRequestWrapper(httpRequest);
        
        RequestContextHolder.setRequestId(httpRequest.getHeader(REQUEST_ID_HEADER));
        RequestContextHolder.setUserId(httpRequest.getHeader(USER_ID_HEADER));
        RequestContextHolder.setUserName(httpRequest.getHeader(USER_NAME_HEADER));
        RequestContextHolder.setRetryFlag(Boolean.parseBoolean(
                httpRequest.getHeader(RETRY_FLAG_HEADER)));

        Map<String, String> headers = new HashMap<>();
        httpRequest.getHeaderNames().asIterator().forEachRemaining(name -> {
            headers.put(name, httpRequest.getHeader(name));
        });
        RequestContextHolder.setRequestHeaders(headers);
        
        chain.doFilter(wrappedRequest, response);
    }
}
```

### 4.2 Controller 层 AOP

#### 4.2.0 技术选型评估

| 方案 | 优点 | 缺点 | 适用场景 |
| ---- | ---- | ---- | -------- |
| **Filter** | 请求进入最早，性能好 | 无法获取方法参数和返回值，无法拦截特定注解 | 全局请求预处理 |
| **Interceptor** | 可以获取请求上下文，支持拦截特定路径 | 无法获取方法参数校验结果 | 请求级别的拦截处理 |
| **AOP Around** | 可以获取完整的方法签名、参数、返回值、异常 | 性能略低 | 日志记录、权限验证、事务管理 |
| **@ControllerAdvice** | 统一异常处理，配合 @Validated 可获取校验结果 | 仅处理异常场景 | 全局异常处理、参数校验结果处理 |

**最终选型**：**AOP Around + @ControllerAdvice 组合**

- **AOP Around**：用于日志记录、幂等处理、权限验证、审计日志
- **@ControllerAdvice**：用于全局异常处理，配合 `@Validated` 注解参数校验结果处理

**选型理由**：
1. AOP 可以完整获取方法执行的上下文（参数、返回值、异常），适合日志记录和业务逻辑增强
2. @ControllerAdvice 可以统一处理 `@Validated` 参数校验失败的异常，与 AOP 配合使用可实现完整的参数校验结果处理

#### 4.2.1 ControllerLogAspect 实现

```java
package com.houyu.common.app.aop.controller;

import com.houyu.common.app.context.RequestContextHolder;
import com.houyu.common.log.model.HyLogEvent;
import com.houyu.common.log.model.HttpRequestInfo;
import com.houyu.common.log.model.HttpResponseInfo;
import com.houyu.common.log.output.LogOutputManager;
import jakarta.servlet.http.HttpServletRequest;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@Aspect
@Component
public class ControllerLogAspect {

    private final LogOutputManager logOutputManager;

    public ControllerLogAspect(LogOutputManager logOutputManager) {
        this.logOutputManager = logOutputManager;
    }

    @Around("execution(* com.houyu.*.controller..*.*(..))")
    public Object logController(ProceedingJoinPoint joinPoint) throws Throwable {
        long startTime = System.currentTimeMillis();
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes != null ? attributes.getRequest() : null;
        HttpServletResponse response = attributes != null ? attributes.getResponse() : null;
        
        String traceId = RequestContextHolder.getTraceId();
        String method = request != null ? request.getMethod() : "";
        String url = request != null ? request.getRequestURI() : "";
        String queryString = request != null ? request.getQueryString() : "";
        
        Map<String, String> requestHeaders = new HashMap<>();
        if (request != null) {
            request.getHeaderNames().asIterator().forEachRemaining(name -> {
                requestHeaders.put(name, request.getHeader(name));
            });
        }

        String requestBody = getRequestBody(request);

        Object result = null;
        Exception exception = null;
        boolean success = true;

        try {
            result = joinPoint.proceed();
            return result;
        } catch (Exception e) {
            exception = e;
            success = false;
            throw e;
        } finally {
            long executionTime = System.currentTimeMillis() - startTime;
            
            HyLogEvent logEvent = new HyLogEvent();
            logEvent.setTraceId(traceId);
            logEvent.setTimestamp(LocalDateTime.now());
            logEvent.setLevel(success ? 
                    com.houyu.common.log.model.LogLevel.WARN : 
                    com.houyu.common.log.model.LogLevel.ERROR);
            logEvent.setMessage(success ? "Controller request success" : "Controller request failed");
            logEvent.setSuccess(success);
            logEvent.setExecutionTime(executionTime);
            logEvent.setServiceName("hy-common-app");
            logEvent.setMethodName(joinPoint.getSignature().getName());
            logEvent.setClassName(joinPoint.getTarget().getClass().getName());

            HttpRequestInfo httpRequest = new HttpRequestInfo();
            httpRequest.setMethod(method);
            httpRequest.setUri(url);
            httpRequest.setQueryString(queryString);
            httpRequest.setHeaders(requestHeaders);
            httpRequest.setRequestBody(requestBody);
            logEvent.setHttpRequest(httpRequest);

            if (exception != null) {
                logEvent.setExceptionClassName(exception.getClass().getName());
                logEvent.setExceptionMessage(exception.getMessage());
            }

            if (result != null || response != null) {
                HttpResponseInfo httpResponse = new HttpResponseInfo();
                if (result != null) {
                    httpResponse.setResponseBody(result.toString());
                }
                
                Map<String, String> responseHeaders = new HashMap<>();
                if (response != null) {
                    response.getHeaderNames().forEach(name -> {
                        responseHeaders.put(name, response.getHeader(name));
                    });
                    httpResponse.setHeaders(responseHeaders);
                    httpResponse.setStatusCode(response.getStatus());
                }
                logEvent.setHttpResponse(httpResponse);
            }

            logOutputManager.output(logEvent);
        }
    }

    private String getRequestBody(HttpServletRequest request) {
        if (request == null) {
            return null;
        }
        
        String contentType = request.getContentType();
        if (contentType != null && (contentType.contains("application/json") || 
                contentType.contains("application/x-www-form-urlencoded"))) {
            if (request instanceof ContentCachingRequestWrapper wrapper) {
                byte[] body = wrapper.getContentAsByteArray();
                if (body != null && body.length > 0) {
                    return new String(body, java.nio.charset.StandardCharsets.UTF_8);
                }
            }
        }
        return null;
    }
}
```

#### 4.2.2 IdempotentAspect 实现

```java
package com.houyu.common.app.aop.controller;

import com.houyu.common.app.annotation.Idempotent;
import com.houyu.common.app.context.RequestContextHolder;
import com.houyu.common.app.enums.IdempotentStatus;
import com.houyu.common.app.service.IdempotentService;
import jakarta.servlet.http.HttpServletRequest;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

@Aspect
@Component
public class IdempotentAspect {

    private static final String REQUEST_ID_HEADER = "X-Request-Id";
    private final IdempotentService idempotentService;

    public IdempotentAspect(IdempotentService idempotentService) {
        this.idempotentService = idempotentService;
    }

    @Around("@annotation(idempotent)")
    public Object handleIdempotent(ProceedingJoinPoint joinPoint, Idempotent idempotent) throws Throwable {
        HttpServletRequest request = getRequest();
        String requestId = request.getHeader(REQUEST_ID_HEADER);
        boolean retryFlag = Boolean.parseBoolean(request.getHeader("X-Retry-Flag"));
        
        if (requestId == null || requestId.isEmpty()) {
            throw new IllegalArgumentException("RequestId is required for idempotent check");
        }

        String key = buildIdempotentKey(requestId, request.getMethod(), request.getRequestURI());
        IdempotentStatus status = idempotentService.getStatus(key);
        
        if (status == IdempotentStatus.PROCESSING) {
            throw new IllegalStateException("Request is processing");
        }
        
        if (!retryFlag && status == IdempotentStatus.SUCCESS) {
            throw new IllegalStateException("Duplicate request detected");
        }

        idempotentService.store(key, IdempotentStatus.PROCESSING);

        try {
            Object result = joinPoint.proceed();
            idempotentService.store(key, IdempotentStatus.SUCCESS);
            return result;
        } catch (Exception e) {
            idempotentService.store(key, IdempotentStatus.FAILED);
            throw e;
        }
    }

    private String buildIdempotentKey(String requestId, String method, String uri) {
        return "idempotent:" + method + ":" + uri + ":" + requestId;
    }

    private HttpServletRequest getRequest() {
        ServletRequestAttributes attributes = 
                (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        return attributes != null ? attributes.getRequest() : null;
    }
}
```

#### 4.2.3 IdempotentService 实现

```java
package com.houyu.common.app.service;

import com.alicp.jetcache.Cache;
import com.alicp.jetcache.anno.CacheType;
import com.alicp.jetcache.anno.CreateCache;
import com.houyu.common.app.enums.IdempotentStatus;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Service
public class IdempotentService {

    @CreateCache(name = "app:idempotent:",
                 cacheType = CacheType.REMOTE,
                 expire = 300,
                 timeUnit = TimeUnit.SECONDS)
    private Cache<String, IdempotentStatus> idempotentCache;

    public IdempotentStatus getStatus(String key) {
        return idempotentCache.get(key);
    }

    public void store(String key, IdempotentStatus status) {
        idempotentCache.put(key, status);
    }

    public void remove(String key) {
        idempotentCache.remove(key);
    }
}
```

#### 4.2.4 IdempotentStatus 枚举

```java
package com.houyu.common.app.enums;

public enum IdempotentStatus {
    PROCESSING,
    SUCCESS,
    FAILED
}
```

#### 4.2.4 AuthAspect 实现

```java
package com.houyu.common.app.aop.controller;

import com.houyu.common.app.annotation.RequirePermission;
import com.houyu.common.app.context.RequestContextHolder;
import com.houyu.common.app.service.PermissionService;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class AuthAspect {

    private final PermissionService permissionService;

    public AuthAspect(PermissionService permissionService) {
        this.permissionService = permissionService;
    }

    @Around("@annotation(requirePermission)")
    public Object checkPermission(ProceedingJoinPoint joinPoint, RequirePermission requirePermission) throws Throwable {
        String userId = RequestContextHolder.getUserId();
        String permission = requirePermission.value();
        
        if (!permissionService.hasPermission(userId, permission)) {
            throw new SecurityException("Insufficient permission: " + permission);
        }

        String dataScope = permissionService.getDataScope(userId);
        RequestContextHolder.setDataScopeSql(dataScope);

        return joinPoint.proceed();
    }
}
```

#### 4.2.5 RequirePermission 注解

```java
package com.houyu.common.app.annotation;

import java.lang.annotation.*;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RequirePermission {
    String value();
    String dataScope() default "";
}
```

#### 4.2.6 AuditAspect 实现

```java
package com.houyu.common.app.aop.controller;

import com.houyu.common.app.annotation.AuditLog;
import com.houyu.common.app.context.RequestContextHolder;
import com.houyu.common.log.model.HyLogEvent;
import com.houyu.common.log.output.LogOutputManager;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;

@Aspect
@Component
public class AuditAspect {

    private final LogOutputManager logOutputManager;

    public AuditAspect(LogOutputManager logOutputManager) {
        this.logOutputManager = logOutputManager;
    }

    @Around("@annotation(auditLog)")
    public Object audit(ProceedingJoinPoint joinPoint, AuditLog auditLog) throws Throwable {
        Exception exception = null;
        boolean success = true;
        
        try {
            return joinPoint.proceed();
        } catch (Exception e) {
            exception = e;
            success = false;
            throw e;
        } finally {
            HyLogEvent logEvent = new HyLogEvent();
            logEvent.setTraceId(RequestContextHolder.getTraceId());
            logEvent.setTimestamp(LocalDateTime.now());
            logEvent.setLevel(success ? 
                    com.houyu.common.log.model.LogLevel.INFO : 
                    com.houyu.common.log.model.LogLevel.ERROR);
            logEvent.setMessage("Audit log: " + auditLog.description() + 
                    (success ? " - success" : " - failed"));
            logEvent.setServiceName("hy-common-app");
            logEvent.setMethodName(joinPoint.getSignature().getName());
            logEvent.setClassName(joinPoint.getTarget().getClass().getName());
            logEvent.setUserId(RequestContextHolder.getUserId());
            logEvent.setSuccess(success);
            
            if (!success && exception != null) {
                logEvent.setExceptionClassName(exception.getClass().getName());
                logEvent.setExceptionMessage(exception.getMessage());
            }
            
            logOutputManager.output(logEvent);
        }
    }
}
```

#### 4.2.7 AuditLog 注解

```java
package com.houyu.common.app.annotation;

import java.lang.annotation.*;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface AuditLog {
    String description();
    String module() default "";
}
```

### 4.3 Manager 层 AOP

#### 4.3.1 ManagerLogAspect 实现

```java
package com.houyu.common.app.aop.manager;

import com.houyu.common.app.context.RequestContextHolder;
import com.houyu.common.log.model.HyLogEvent;
import com.houyu.common.log.output.LogOutputManager;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.util.Arrays;

@Aspect
@Component
public class ManagerLogAspect {

    private final LogOutputManager logOutputManager;

    public ManagerLogAspect(LogOutputManager logOutputManager) {
        this.logOutputManager = logOutputManager;
    }

    @Around("execution(* com.houyu.*.manager..*.*(..))")
    public Object logManager(ProceedingJoinPoint joinPoint) throws Throwable {
        long startTime = System.currentTimeMillis();
        String traceId = RequestContextHolder.getTraceId();
        String methodSignature = joinPoint.getSignature().toShortString();
        String className = joinPoint.getTarget().getClass().getName();
        Object[] args = joinPoint.getArgs();

        Object result = null;
        Exception exception = null;
        boolean success = true;

        try {
            result = joinPoint.proceed();
            return result;
        } catch (Exception e) {
            exception = e;
            success = false;
            throw e;
        } finally {
            long executionTime = System.currentTimeMillis() - startTime;

            HyLogEvent logEvent = new HyLogEvent();
            logEvent.setTraceId(traceId);
            logEvent.setTimestamp(LocalDateTime.now());
            logEvent.setLevel(success ? 
                    com.houyu.common.log.model.LogLevel.INFO : 
                    com.houyu.common.log.model.LogLevel.ERROR);
            logEvent.setMessage(success ? "Manager method success" : "Manager method failed");
            logEvent.setSuccess(success);
            logEvent.setExecutionTime(executionTime);
            logEvent.setServiceName("hy-common-app");
            logEvent.setMethodName(methodSignature);
            logEvent.setClassName(className);
            logEvent.setMdcContext(new java.util.HashMap<>(RequestContextHolder.getMdcContext()));

            if (args != null && args.length > 0) {
                logEvent.setMessage("Args: " + Arrays.toString(args));
            }

            if (result != null) {
                logEvent.setMessage(logEvent.getMessage() + ", Result: " + result.toString());
            }

            if (exception != null) {
                logEvent.setExceptionClassName(exception.getClass().getName());
                logEvent.setExceptionMessage(exception.getMessage());
            }

            logOutputManager.output(logEvent);
        }
    }
}
```

> **说明**：Manager 层事务管理不再通过 TransactionAspect 统一处理，改为在具体写操作方法上显式标注 `@Transactional(rollbackFor = Exception.class)` 注解。

### 4.4 Service 层 AOP

#### 4.4.1 ServiceLogAspect 实现

```java
package com.houyu.common.app.aop.service;

import com.houyu.common.app.context.RequestContextHolder;
import com.houyu.common.log.model.HyLogEvent;
import com.houyu.common.log.output.LogOutputManager;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.util.Arrays;

@Aspect
@Component
public class ServiceLogAspect {

    private final LogOutputManager logOutputManager;

    public ServiceLogAspect(LogOutputManager logOutputManager) {
        this.logOutputManager = logOutputManager;
    }

    @Around("execution(* com.houyu.*.service..*.*(..))")
    public Object logService(ProceedingJoinPoint joinPoint) throws Throwable {
        long startTime = System.currentTimeMillis();
        String traceId = RequestContextHolder.getTraceId();
        String methodName = joinPoint.getSignature().getName();
        String className = joinPoint.getTarget().getClass().getName();
        Object[] args = joinPoint.getArgs();

        Object result = null;
        Exception exception = null;
        boolean success = true;

        try {
            result = joinPoint.proceed();
            return result;
        } catch (Exception e) {
            exception = e;
            success = false;
            throw e;
        } finally {
            long executionTime = System.currentTimeMillis() - startTime;

            HyLogEvent logEvent = new HyLogEvent();
            logEvent.setTraceId(traceId);
            logEvent.setTimestamp(LocalDateTime.now());
            logEvent.setLevel(success ? 
                    com.houyu.common.log.model.LogLevel.DEBUG : 
                    com.houyu.common.log.model.LogLevel.ERROR);
            logEvent.setMessage(success ? "Service method success" : "Service method failed");
            logEvent.setSuccess(success);
            logEvent.setExecutionTime(executionTime);
            logEvent.setServiceName("hy-common-app");
            logEvent.setMethodName(methodName);
            logEvent.setClassName(className);

            if (args != null && args.length > 0) {
                logEvent.setMessage("Args: " + Arrays.toString(args));
            }

            if (result != null) {
                logEvent.setMessage(logEvent.getMessage() + ", Result: " + result.toString());
            }

            if (exception != null) {
                logEvent.setExceptionClassName(exception.getClass().getName());
                logEvent.setExceptionMessage(exception.getMessage());
            }

            logOutputManager.output(logEvent);
        }
    }
}
```

#### 4.4.2 EntityFillAspect 实现

```java
package com.houyu.common.app.aop.service;

import com.baomidou.mybatisplus.core.conditions.Wrapper;
import com.houyu.common.app.context.RequestContextHolder;
import com.houyu.common.app.entity.BaseEntity;
import com.houyu.common.app.enums.OpType;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.util.Collection;

@Aspect
@Component
public class EntityFillAspect {

    @Before("execution(* com.houyu.*.service..*.save*(..))")
    public void fillForSave(JoinPoint joinPoint) {
        fillEntity(joinPoint, OpType.INSERT);
    }

    @Before("execution(* com.houyu.*.service..*.update*(..))")
    public void fillForUpdate(JoinPoint joinPoint) {
        fillEntity(joinPoint, OpType.UPDATE);
    }

    @Before("execution(* com.houyu.*.service..*.remove*(..))")
    public void fillForDelete(JoinPoint joinPoint) {
        fillEntity(joinPoint, OpType.DELETE);
    }

    @Before("execution(* com.houyu.*.service..*.saveOrUpdate*(..))")
    public void fillForSaveOrUpdate(JoinPoint joinPoint) {
        fillEntity(joinPoint, OpType.INSERT);
    }

    private void fillEntity(JoinPoint joinPoint, OpType opType) {
        String traceId = RequestContextHolder.getTraceId();
        String userName = RequestContextHolder.getUserName();

        for (Object arg : joinPoint.getArgs()) {
            if (arg instanceof BaseEntity entity) {
                fillBaseEntity(entity, traceId, userName, opType);
            } else if (arg instanceof Collection<?> collection) {
                collection.forEach(item -> {
                    if (item instanceof BaseEntity entity) {
                        fillBaseEntity(entity, traceId, userName, opType);
                    }
                });
            } else if (arg instanceof Wrapper<?>) {
                Object entity = extractEntityFromWrapper(arg);
                if (entity instanceof BaseEntity baseEntity) {
                    fillBaseEntity(baseEntity, traceId, userName, opType);
                }
            }
        }
    }

    private void fillBaseEntity(BaseEntity entity, String traceId, String userName, OpType opType) {
        entity.setTraceId(traceId);
        entity.setOpType(opType.name());
        entity.setOwner(userName);
        
        if (opType == OpType.INSERT) {
            entity.setCreater(userName);
            entity.setCreateTime(LocalDateTime.now());
        }
        
        entity.setUpdater(userName);
        entity.setUpdateTime(LocalDateTime.now());
        entity.setIsCurrent(true);
    }

    private Object extractEntityFromWrapper(Object wrapper) {
        try {
            java.lang.reflect.Field entityField = wrapper.getClass().getDeclaredField("entity");
            entityField.setAccessible(true);
            return entityField.get(wrapper);
        } catch (Exception e) {
            return null;
        }
    }
}
```

#### 4.4.3 JournalAspect 实现

```java
package com.houyu.common.app.aop.service;

import com.houyu.common.app.context.RequestContextHolder;
import com.houyu.common.app.entity.BaseEntity;
import com.houyu.common.app.service.JournalService;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class JournalAspect {

    private final JournalService journalService;

    public JournalAspect(JournalService journalService) {
        this.journalService = journalService;
    }

    @Around("execution(* com.houyu.*.service..*.save*(..))")
    public Object recordSaveJournal(ProceedingJoinPoint joinPoint) throws Throwable {
        Object result = joinPoint.proceed();
        
        for (Object arg : joinPoint.getArgs()) {
            if (arg instanceof BaseEntity entity) {
                journalService.sendJournal(entity);
            }
        }
        
        return result;
    }

    @Around("execution(* com.houyu.*.service..*.update*(..))")
    public Object recordUpdateJournal(ProceedingJoinPoint joinPoint) throws Throwable {
        Object result = joinPoint.proceed();
        
        for (Object arg : joinPoint.getArgs()) {
            if (arg instanceof BaseEntity entity) {
                journalService.sendJournal(entity);
            }
        }
        
        return result;
    }

    @Around("execution(* com.houyu.*.service..*.remove*(..))")
    public Object recordDeleteJournal(ProceedingJoinPoint joinPoint) throws Throwable {
        Object[] args = joinPoint.getArgs();
        
        BaseEntity entity = extractEntityFromArgs(args);
        
        Object result = joinPoint.proceed();
        
        if (entity != null) {
            journalService.sendJournal(entity);
        }
        
        return result;
    }

    private BaseEntity extractEntityFromArgs(Object[] args) {
        for (Object arg : args) {
            if (arg instanceof BaseEntity) {
                return (BaseEntity) arg;
            }
        }
        return null;
    }
}
```

#### 4.4.4 JournalService 实现

```java
package com.houyu.common.app.service;

import com.houyu.common.app.entity.BaseEntity;
import org.springframework.stereotype.Service;

@Service
public class JournalService {

    public void sendJournal(BaseEntity entity) {
        BaseEntity journalEntity = cloneEntity(entity);
        journalEntity.setOpType(entity.getOpType());
        journalEntity.setTraceId(entity.getTraceId());
        
        // 发送消息到消息队列，异步写入流水表
        // 实际实现可使用 Kafka/RabbitMQ 等消息中间件
        System.out.println("Sending journal for entity: " + entity.getClass().getSimpleName());
    }

    private BaseEntity cloneEntity(BaseEntity entity) {
        try {
            return entity.getClass().getDeclaredConstructor().newInstance();
        } catch (Exception e) {
            return null;
        }
    }
}
```

#### 4.4.5 OpType 枚举

```java
package com.houyu.common.app.enums;

public enum OpType {
    INSERT,
    UPDATE,
    DELETE
}
```

### 4.5 Mapper 层拦截器

#### 4.5.1 TableShardInterceptor 实现

```java
package com.houyu.common.app.mybatis;

import com.houyu.common.app.entity.ShardEntity;
import org.apache.ibatis.executor.statement.StatementHandler;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.plugin.*;
import org.apache.ibatis.reflection.DefaultReflectorFactory;
import org.apache.ibatis.reflection.MetaObject;
import org.apache.ibatis.reflection.SystemMetaObject;

import java.sql.Connection;
import java.util.Properties;

@Intercepts({@Signature(
        type = StatementHandler.class,
        method = "prepare",
        args = {Connection.class, Integer.class})})
public class TableShardInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        StatementHandler statementHandler = (StatementHandler) invocation.getTarget();
        MetaObject metaObject = MetaObject.forObject(
                statementHandler,
                SystemMetaObject.DEFAULT_OBJECT_FACTORY,
                SystemMetaObject.DEFAULT_OBJECT_WRAPPER_FACTORY,
                new DefaultReflectorFactory());

        MappedStatement mappedStatement = 
                (MappedStatement) metaObject.getValue("delegate.mappedStatement");
        
        String sql = (String) metaObject.getValue("delegate.boundSql.sql");
        
        Object parameterObject = metaObject.getValue("delegate.boundSql.parameterObject");
        if (parameterObject instanceof ShardEntity shardEntity) {
            String tableNameSrc = shardEntity.getTableNameSrc();
            String tableNameDest = shardEntity.getTableNameDest();
            
            if (tableNameSrc != null && tableNameDest != null) {
                sql = sql.replace(tableNameSrc, tableNameDest);
                metaObject.setValue("delegate.boundSql.sql", sql);
            }
        }

        return invocation.proceed();
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
    }
}
```

#### 4.5.2 DataPermissionInterceptor 实现

```java
package com.houyu.common.app.mybatis;

import com.houyu.common.app.context.RequestContextHolder;
import org.apache.ibatis.executor.statement.StatementHandler;
import org.apache.ibatis.plugin.*;

import java.sql.Connection;
import java.util.Properties;

@Intercepts({@Signature(
        type = StatementHandler.class,
        method = "prepare",
        args = {Connection.class, Integer.class})})
public class DataPermissionInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        StatementHandler statementHandler = (StatementHandler) invocation.getTarget();
        
        String dataScopeSql = RequestContextHolder.getDataScopeSql();
        if (dataScopeSql != null && !dataScopeSql.isEmpty()) {
            String originalSql = statementHandler.getBoundSql().getSql();
            if (originalSql.toUpperCase().contains("SELECT")) {
                String newSql = originalSql + " " + dataScopeSql;
                com.houyu.common.app.util.SqlUtils.setSql(statementHandler, newSql);
            }
        }

        return invocation.proceed();
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
    }
}
```

#### 4.5.3 PageInterceptor 实现

```java
package com.houyu.common.app.mybatis;

import com.baomidou.mybatisplus.core.metadata.IPage;
import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.plugin.*;
import org.apache.ibatis.session.ResultHandler;
import org.apache.ibatis.session.RowBounds;

import java.util.Properties;

@Intercepts({@Signature(
        type = Executor.class,
        method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})})
public class PageInterceptor implements Interceptor {

    private static final int MAX_PAGE_SIZE = 5000;

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object parameter = invocation.getArgs()[1];
        
        if (parameter instanceof IPage<?> page) {
            long size = page.getSize();
            if (size > MAX_PAGE_SIZE) {
                page.setSize(MAX_PAGE_SIZE);
            }
        }
        
        return invocation.proceed();
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
    }
}
```

> **说明**：PageInterceptor 与 PaginationInnerInterceptor 注册顺序说明：
> 1. PageInterceptor 应在 PaginationInnerInterceptor 之前注册（order 值更小）
> 2. PageInterceptor 负责截断 size 值，保证安全
> 3. PaginationInnerInterceptor 负责生成分页 SQL
> 4. 在 MyBatisPlusConfig 中配置时，PageInterceptor 的 addInterceptor 调用应早于 PaginationInnerInterceptor

#### 4.5.4 SQL 日志记录 - p6spy 集成

> **说明**：不再使用 SqlLogInterceptor，改为集成 p6spy 实现 SQL 日志记录。p6spy 是一个 JDBC 代理，能够完整记录执行的 SQL 语句（已完成参数替换），比 MyBatis 拦截器方式更可靠。

**pom.xml 依赖**：
```xml
<dependency>
    <groupId>p6spy</groupId>
    <artifactId>p6spy</artifactId>
    <version>3.9.1</version>
</dependency>
```

**spy.properties 配置**：
```properties
modulelist=com.p6spy.engine.spy.P6SpyModule,com.p6spy.engine.logging.P6LogFactory
appender=com.p6spy.engine.spy.appender.Slf4JLogger
logMessageFormat=com.p6spy.engine.spy.appender.CustomLineFormat
customLogMessageFormat=%(currentTime) | %(executionTime)ms | %(category) | connection%(connectionId) | %(sqlSingleLine)
dateformat=yyyy-MM-dd HH:mm:ss
```

**数据源配置**：
```yaml
spring:
  datasource:
    url: jdbc:p6spy:postgresql://localhost:5432/postgres
    driver-class-name: com.p6spy.engine.spy.P6SpyDriver
```

### 4.6 公共 Entity

#### 4.6.1 BaseEntity 实现

```java
package com.houyu.common.app.entity;

import com.baomidou.mybatisplus.annotation.FieldFill;
import com.baomidou.mybatisplus.annotation.IdType;
import com.baomidou.mybatisplus.annotation.TableField;
import com.baomidou.mybatisplus.annotation.TableId;
import com.fasterxml.jackson.annotation.JsonFormat;
import io.swagger.v3.oas.annotations.media.Schema;
import lombok.Data;

import java.io.Serializable;
import java.time.LocalDateTime;

@Data
@Schema(description = "公共实体基类")
public class BaseEntity implements Serializable {

    private static final long serialVersionUID = 1L;

    @Schema(description = "自增主键（内部排序，禁止插队）")
    @TableId(value = "id", type = IdType.AUTO)
    private Long id;

    @Schema(description = "是否当前有效版本")
    @TableField(value = "is_current")
    private Boolean isCurrent;

    @Schema(description = "日志链路追踪ID")
    @TableField(value = "trace_id", fill = FieldFill.INSERT_UPDATE)
    private String traceId;

    @Schema(description = "数据拥有人")
    @TableField(value = "owner", fill = FieldFill.INSERT_UPDATE)
    private String owner;

    @Schema(description = "创建人")
    @TableField(value = "creater", fill = FieldFill.INSERT)
    private String creater;

    @Schema(description = "创建时间")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    @TableField(value = "create_time", fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @Schema(description = "修改人")
    @TableField(value = "updater", fill = FieldFill.INSERT_UPDATE)
    private String updater;

    @Schema(description = "修改时间")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    @TableField(value = "update_time", fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;

    @Schema(description = "操作类型（insert,update,delete）")
    @TableField(value = "op_type", fill = FieldFill.INSERT_UPDATE)
    private String opType;

    @Schema(description = "备注")
    @TableField(value = "remark")
    private String remark;
}
```

#### 4.6.2 ShardEntity 实现

```java
package com.houyu.common.app.entity;

import io.swagger.v3.oas.annotations.media.Schema;
import lombok.Data;
import lombok.EqualsAndHashCode;

@Data
@EqualsAndHashCode(callSuper = true)
@Schema(description = "公共分表实体")
public class ShardEntity extends BaseEntity {

    @Schema(description = "替换前表名")
    private String tableNameSrc;

    @Schema(description = "替换后表名")
    private String tableNameDest;
}
```

### 4.7 RequestContextHolder 实现

```java
package com.houyu.common.app.context;

import com.alibaba.ttl.TransmittableThreadLocal;

import java.util.HashMap;
import java.util.Map;

public class RequestContextHolder {

    private static final TransmittableThreadLocal<String> traceIdHolder = new TransmittableThreadLocal<>();
    private static final TransmittableThreadLocal<String> requestIdHolder = new TransmittableThreadLocal<>();
    private static final TransmittableThreadLocal<String> userIdHolder = new TransmittableThreadLocal<>();
    private static final TransmittableThreadLocal<String> userNameHolder = new TransmittableThreadLocal<>();
    private static final TransmittableThreadLocal<Boolean> retryFlagHolder = new TransmittableThreadLocal<>();
    private static final TransmittableThreadLocal<String> dataScopeSqlHolder = new TransmittableThreadLocal<>();
    private static final TransmittableThreadLocal<Map<String, String>> requestHeadersHolder = new TransmittableThreadLocal<>();
    private static final TransmittableThreadLocal<Map<String, String>> mdcContextHolder = new TransmittableThreadLocal<>();

    public static void setTraceId(String traceId) {
        traceIdHolder.set(traceId);
        updateMdc("traceId", traceId);
    }

    public static String getTraceId() {
        return traceIdHolder.get();
    }

    public static void setRequestId(String requestId) {
        requestIdHolder.set(requestId);
        updateMdc("requestId", requestId);
    }

    public static String getRequestId() {
        return requestIdHolder.get();
    }

    public static void setUserId(String userId) {
        userIdHolder.set(userId);
        updateMdc("userId", userId);
    }

    public static String getUserId() {
        return userIdHolder.get();
    }

    public static void setUserName(String userName) {
        userNameHolder.set(userName);
        updateMdc("userName", userName);
    }

    public static String getUserName() {
        return userNameHolder.get();
    }

    public static void setRetryFlag(Boolean retryFlag) {
        retryFlagHolder.set(retryFlag);
    }

    public static Boolean getRetryFlag() {
        return retryFlagHolder.get();
    }

    public static void setDataScopeSql(String dataScopeSql) {
        dataScopeSqlHolder.set(dataScopeSql);
    }

    public static String getDataScopeSql() {
        return dataScopeSqlHolder.get();
    }

    public static void setRequestHeaders(Map<String, String> headers) {
        requestHeadersHolder.set(headers);
    }

    public static Map<String, String> getRequestHeaders() {
        return requestHeadersHolder.get();
    }

    public static Map<String, String> getMdcContext() {
        return mdcContextHolder.get() != null ? mdcContextHolder.get() : new HashMap<>();
    }

    private static void updateMdc(String key, String value) {
        Map<String, String> mdc = mdcContextHolder.get();
        if (mdc == null) {
            mdc = new HashMap<>();
            mdcContextHolder.set(mdc);
        }
        mdc.put(key, value);
    }

    public static void clear() {
        traceIdHolder.remove();
        requestIdHolder.remove();
        userIdHolder.remove();
        userNameHolder.remove();
        retryFlagHolder.remove();
        dataScopeSqlHolder.remove();
        requestHeadersHolder.remove();
        mdcContextHolder.remove();
    }
}
```

## 5. Maven 依赖配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.houyu</groupId>
        <artifactId>hy-common</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>

    <groupId>com.houyu</groupId>
    <artifactId>hy-common-app</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>
    <name>hy-common-app</name>
    <description>HouYu Common App Module - 后端应用公共模块，提供AOP切面、拦截器、公共Entity等</description>

    <dependencies>
        <!-- Spring Boot Starter Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Spring Boot Starter AOP -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>

        <!-- Spring Boot Starter Validation -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- MyBatis Plus -->
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot3-starter</artifactId>
            <version>3.5.6</version>
        </dependency>

        <!-- SpringDoc OpenAPI (Swagger) -->
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>2.3.0</version>
        </dependency>

        <!-- JetCache -->
        <dependency>
            <groupId>com.alicp.jetcache</groupId>
            <artifactId>jetcache-starter-redis</artifactId>
            <version>${jetcache.version}</version>
        </dependency>

        <!-- Transmittable Thread Local -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>transmittable-thread-local</artifactId>
            <version>${transmittable-thread-local.version}</version>
        </dependency>

        <!-- hy-common-log -->
        <dependency>
            <groupId>com.houyu</groupId>
            <artifactId>hy-common-log</artifactId>
        </dependency>

        <!-- p6spy for SQL logging -->
        <dependency>
            <groupId>p6spy</groupId>
            <artifactId>p6spy</artifactId>
            <version>3.9.1</version>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>

</project>
```

## 6. 配置说明

### 6.1 application.yml 配置示例

```yaml
hy:
  app:
    enabled: true
    idempotent:
      expire-seconds: 300
    page:
      max-size: 5000
    security:
      enabled: true

spring:
  application:
    name: hy-common-app
  cloud:
    nacos:
      discovery:
        server-addr: ${NACOS_SERVER_ADDR:localhost:8848}
      config:
        server-addr: ${NACOS_SERVER_ADDR:localhost:8848}

jetcache:
  local:
    default:
      type: caffeine
      limit: 10000
  remote:
    default:
      type: redis.lettuce
      keyConvertor: fastjson2
      uri: redis://localhost:6379

mybatis-plus:
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

### 6.2 AppProperties 配置类

```java
package com.houyu.common.app.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Data
@Component
@ConfigurationProperties(prefix = "hy.app")
public class AppProperties {

    private boolean enabled = true;
    
    private IdempotentConfig idempotent = new IdempotentConfig();
    private PageConfig page = new PageConfig();
    private SecurityConfig security = new SecurityConfig();

    @Data
    public static class IdempotentConfig {
        private int expireSeconds = 300;
    }

    @Data
    public static class PageConfig {
        private int maxSize = 5000;
    }

    @Data
    public static class SecurityConfig {
        private boolean enabled = true;
    }
}
```

### 6.3 AutoConfiguration.imports 配置

**文件路径**：`src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

```
com.houyu.common.app.config.AppAutoConfiguration
com.houyu.common.app.config.MyBatisPlusConfig
com.houyu.common.app.interceptor.TraceFilter
com.houyu.common.app.interceptor.RequestFilter
com.houyu.common.app.aop.controller.ControllerLogAspect
com.houyu.common.app.aop.controller.IdempotentAspect
com.houyu.common.app.aop.controller.AuthAspect
com.houyu.common.app.aop.controller.AuditAspect
com.houyu.common.app.aop.manager.ManagerLogAspect
com.houyu.common.app.aop.service.ServiceLogAspect
com.houyu.common.app.aop.service.EntityFillAspect
com.houyu.common.app.aop.service.JournalAspect
com.houyu.common.app.mybatis.TableShardInterceptor
com.houyu.common.app.mybatis.DataPermissionInterceptor
com.houyu.common.app.mybatis.PageInterceptor
```

### 6.4 MyBatisPlusConfig 配置类

```java
package com.houyu.common.app.config;

import com.baomidou.mybatisplus.annotation.DbType;
import com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor;
import com.baomidou.mybatisplus.extension.plugins.inner.PaginationInnerInterceptor;
import com.houyu.common.app.mybatis.DataPermissionInterceptor;
import com.houyu.common.app.mybatis.PageInterceptor;
import com.houyu.common.app.mybatis.TableShardInterceptor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MyBatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        
        interceptor.addInnerInterceptor(new PageInterceptor());
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.POSTGRE_SQL));
        interceptor.addInnerInterceptor(new TableShardInterceptor());
        interceptor.addInnerInterceptor(new DataPermissionInterceptor());
        
        return interceptor;
    }
}
```

## 7. 异常处理

### 7.1 异常类型

| 异常码 | 异常信息 | HTTP状态码 |
| ------ | -------- | ---------- |
| IDENTITY_MISSING | RequestId is required | 400 |
| DUPLICATE_REQUEST | Duplicate request detected | 409 |
| PERMISSION_DENIED | Insufficient permission | 403 |

## 8. 监控指标

| 指标名称 | 类型 | 说明 |
| -------- | ---- | ---- |
| hy.app.controller.requests.total | Counter | Controller 请求总数 |
| hy.app.controller.idempotent.hits | Counter | 幂等校验命中数 |
| hy.app.controller.auth.failed | Counter | 权限校验失败数 |
| hy.app.service.entity.filled | Counter | Entity 字段填充次数 |
| hy.app.service.journal.sent | Counter | 流水消息发送数 |
| hy.app.mybatis.shard.hits | Counter | 分表拦截次数 |
| hy.app.mybatis.permission.hits | Counter | 数据权限拦截次数 |
| hy.app.mybatis.page.limited | Counter | 分页大小限制触发次数 |

## 9. 安全考虑

1. **敏感信息保护**：日志中不记录完整的请求体和响应体，对敏感字段进行脱敏
2. **权限验证**：通过 `@RequirePermission` 注解进行细粒度权限控制
3. **幂等性保护**：防止重复请求导致数据重复处理
4. **SQL 注入防护**：MyBatis 参数化查询，避免 SQL 注入
5. **数据权限隔离**：通过数据权限拦截器实现行级数据隔离