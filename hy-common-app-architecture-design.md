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
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │ TraceFilter  │→│ ValidFilter  │→│ RequestFilter│              │   │
│  │  │ 验证traceId  │  │ 参数校验     │  │ 请求预处理   │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
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
| 拦截器层 | TraceFilter | 验证请求头中的 traceId，不存在则生成 |
| 拦截器层 | RequestFilter | 请求预处理，设置线程上下文 |
| AOP层 | ControllerLogAspect | Controller 日志记录（warn/error级别） |
| AOP层 | IdempotentAspect | 接口幂等处理 |
| AOP层 | AuthAspect | 权限验证（功能权限、数据权限） |
| AOP层 | AuditAspect | 审计日志记录 |
| AOP层 | ManagerLogAspect | Manager 日志记录（info/error级别） |
| AOP层 | TransactionAspect | 事务管理 |
| AOP层 | ServiceLogAspect | Service 日志记录（debug/error级别） |
| AOP层 | EntityFillAspect | Entity 字段自动赋值 |
| AOP层 | JournalAspect | 流水表记录 |
| 拦截器层 | TableShardInterceptor | MyBatis 分表拦截器 |
| 拦截器层 | DataPermissionInterceptor | 数据权限拦截器 |
| 拦截器层 | PageInterceptor | 分页大小限制拦截器 |
| 拦截器层 | SqlLogInterceptor | SQL 日志记录拦截器 |
| 实体层 | BaseEntity | 公共实体基类 |
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
│   │   │       │   └── AppProperties.java
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
│   │   │       │   │   ├── ManagerLogAspect.java
│   │   │       │   │   └── TransactionAspect.java
│   │   │       │   └── service/
│   │   │       │       ├── ServiceLogAspect.java
│   │   │       │       ├── EntityFillAspect.java
│   │   │       │       └── JournalAspect.java
│   │   │       ├── mybatis/
│   │   │       │   ├── TableShardInterceptor.java
│   │   │       │   ├── DataPermissionInterceptor.java
│   │   │       │   ├── PageInterceptor.java
│   │   │       │   └── SqlLogInterceptor.java
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
│   │   │       │   └── OpType.java
│   │   │       ├── service/
│   │   │       │   ├── IdempotentService.java
│   │   │       │   ├── PermissionService.java
│   │   │       │   └── JournalService.java
│   │   │       └── util/
│   │   │           └── SqlUtils.java
│   │   └── resources/
│   │       └── META-INF/
│   │           └── spring/
│   │               └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
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
| TraceFilter | 验证/生成 traceId，存入线程上下文 | interceptor |
| RequestFilter | 请求预处理，解析请求头信息 | interceptor |
| ControllerLogAspect | 记录 Controller 层日志（warn/error） | aop.controller |
| IdempotentAspect | 基于 requestId 实现接口幂等 | aop.controller |
| AuthAspect | 权限验证，支持注解和全局配置 | aop.controller |
| AuditAspect | 审计日志记录 | aop.controller |
| ManagerLogAspect | 记录 Manager 层日志（info/error） | aop.manager |
| TransactionAspect | 声明式事务管理 | aop.manager |
| ServiceLogAspect | 记录 Service 层日志（debug/error） | aop.service |
| EntityFillAspect | 自动填充 entity 审计字段 | aop.service |
| JournalAspect | 发送消息到流水表 | aop.service |
| TableShardInterceptor | MyBatis 分表拦截器 | mybatis |
| DataPermissionInterceptor | 数据权限 SQL 改写 | mybatis |
| PageInterceptor | 分页大小限制（最大5000） | mybatis |
| SqlLogInterceptor | SQL 日志记录 | mybatis |
| BaseEntity | 公共实体基类，包含审计字段 | entity |
| ShardEntity | 分表实体基类 | entity |
| RequestContextHolder | 线程上下文管理器 | context |

## 4. 核心功能设计

### 4.1 统一拦截器

#### 4.1.1 TraceFilter 实现

```java
package com.houyu.common.app.interceptor;

import com.houyu.common.app.context.RequestContextHolder;
import com.houyu.common.log.trace.TraceIdGenerator;
import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class TraceFilter implements Filter {

    private static final String TRACE_ID_HEADER = "X-Trace-Id";
    private final TraceIdGenerator traceIdGenerator;

    public TraceFilter(TraceIdGenerator traceIdGenerator) {
        this.traceIdGenerator = traceIdGenerator;
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        
        String traceId = httpRequest.getHeader(TRACE_ID_HEADER);
        if (traceId == null || traceId.isEmpty()) {
            traceId = traceIdGenerator.generateTraceId();
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

        try {
            chain.doFilter(request, response);
        } finally {
            // 保留 traceId，供后续日志使用
            String traceId = RequestContextHolder.getTraceId();
            RequestContextHolder.clear();
            RequestContextHolder.setTraceId(traceId);
        }
    }
}
```

### 4.2 Controller 层 AOP

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
        HttpServletRequest request = getRequest();
        String traceId = RequestContextHolder.getTraceId();
        String method = request.getMethod();
        String url = request.getRequestURI();
        String queryString = request.getQueryString();
        
        Map<String, String> headers = new HashMap<>();
        request.getHeaderNames().asIterator().forEachRemaining(name -> {
            headers.put(name, request.getHeader(name));
        });

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
            httpRequest.setHeaders(headers);
            logEvent.setHttpRequest(httpRequest);

            if (exception != null) {
                logEvent.setExceptionClassName(exception.getClass().getName());
                logEvent.setExceptionMessage(exception.getMessage());
            }

            if (result != null) {
                HttpResponseInfo httpResponse = new HttpResponseInfo();
                httpResponse.setResponseBody(result.toString());
                logEvent.setHttpResponse(httpResponse);
            }

            logOutputManager.output(logEvent);
        }
    }

    private HttpServletRequest getRequest() {
        ServletRequestAttributes attributes = 
                (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        return attributes != null ? attributes.getRequest() : null;
    }
}
```

#### 4.2.2 IdempotentAspect 实现

```java
package com.houyu.common.app.aop.controller;

import com.houyu.common.app.annotation.Idempotent;
import com.houyu.common.app.context.RequestContextHolder;
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

    @Around("@annotation(idempotent) || execution(* com.houyu.*.controller..*.*(..))")
    public Object handleIdempotent(ProceedingJoinPoint joinPoint, Idempotent idempotent) throws Throwable {
        HttpServletRequest request = getRequest();
        String requestId = request.getHeader(REQUEST_ID_HEADER);
        boolean retryFlag = Boolean.parseBoolean(request.getHeader("X-Retry-Flag"));
        
        if (requestId == null || requestId.isEmpty()) {
            throw new IllegalArgumentException("RequestId is required for idempotent check");
        }

        String key = buildIdempotentKey(requestId, request.getMethod(), request.getRequestURI());
        
        if (!retryFlag && idempotentService.exists(key)) {
            throw new IllegalStateException("Duplicate request detected");
        }

        idempotentService.store(key);

        try {
            return joinPoint.proceed();
        } finally {
            if (!retryFlag) {
                idempotentService.remove(key);
            }
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
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Service
public class IdempotentService {

    @CreateCache(name = "app:idempotent:",
                 cacheType = CacheType.REMOTE,
                 expire = 300,
                 timeUnit = TimeUnit.SECONDS)
    private Cache<String, Boolean> idempotentCache;

    public boolean exists(String key) {
        return idempotentCache.get(key) != null;
    }

    public void store(String key) {
        idempotentCache.put(key, true);
    }

    public void remove(String key) {
        idempotentCache.remove(key);
    }
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
        Object result = joinPoint.proceed();
        
        HyLogEvent logEvent = new HyLogEvent();
        logEvent.setTraceId(RequestContextHolder.getTraceId());
        logEvent.setTimestamp(LocalDateTime.now());
        logEvent.setLevel(com.houyu.common.log.model.LogLevel.INFO);
        logEvent.setMessage("Audit log: " + auditLog.description());
        logEvent.setServiceName("hy-common-app");
        logEvent.setMethodName(joinPoint.getSignature().getName());
        logEvent.setClassName(joinPoint.getTarget().getClass().getName());
        logEvent.setUserId(RequestContextHolder.getUserId());
        
        logOutputManager.output(logEvent);
        
        return result;
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
                    com.houyu.common.log.model.LogLevel.INFO : 
                    com.houyu.common.log.model.LogLevel.ERROR);
            logEvent.setMessage(success ? "Manager method success" : "Manager method failed");
            logEvent.setSuccess(success);
            logEvent.setExecutionTime(executionTime);
            logEvent.setServiceName("hy-common-app");
            logEvent.setMethodName(methodName);
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

#### 4.3.2 TransactionAspect 实现

```java
package com.houyu.common.app.aop.manager;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

@Aspect
@Component
public class TransactionAspect {

    @Around("execution(* com.houyu.*.manager..*.*(..))")
    @Transactional(rollbackFor = Exception.class)
    public Object manageTransaction(ProceedingJoinPoint joinPoint) throws Throwable {
        return joinPoint.proceed();
    }
}
```

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

import com.houyu.common.app.context.RequestContextHolder;
import com.houyu.common.app.entity.BaseEntity;
import com.houyu.common.app.enums.OpType;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;

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

    private void fillEntity(JoinPoint joinPoint, OpType opType) {
        String traceId = RequestContextHolder.getTraceId();
        String userName = RequestContextHolder.getUserName();

        for (Object arg : joinPoint.getArgs()) {
            if (arg instanceof BaseEntity entity) {
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

    @Around("execution(* com.houyu.*.service..*.save*(..)) || " +
            "execution(* com.houyu.*.service..*.update*(..)) || " +
            "execution(* com.houyu.*.service..*.remove*(..))")
    public Object recordJournal(ProceedingJoinPoint joinPoint) throws Throwable {
        Object result = joinPoint.proceed();
        
        for (Object arg : joinPoint.getArgs()) {
            if (arg instanceof BaseEntity entity) {
                journalService.sendJournal(entity);
            }
        }
        
        return result;
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
        // 发送消息到消息队列，异步写入流水表
        // 实际实现可使用 Kafka/RabbitMQ 等消息中间件
        System.out.println("Sending journal for entity: " + entity.getClass().getSimpleName());
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

import org.apache.ibatis.executor.statement.StatementHandler;
import org.apache.ibatis.plugin.*;

import java.sql.Connection;
import java.util.Properties;

@Intercepts({@Signature(
        type = StatementHandler.class,
        method = "prepare",
        args = {Connection.class, Integer.class})})
public class PageInterceptor implements Interceptor {

    private static final int MAX_PAGE_SIZE = 5000;

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        StatementHandler statementHandler = (StatementHandler) invocation.getTarget();
        String sql = statementHandler.getBoundSql().getSql();
        
        String upperSql = sql.toUpperCase();
        int limitIndex = upperSql.indexOf("LIMIT");
        
        if (limitIndex != -1) {
            int afterLimit = limitIndex + 5;
            int endIndex = upperSql.indexOf(" ", afterLimit);
            if (endIndex == -1) {
                endIndex = upperSql.length();
            }
            
            try {
                int limit = Integer.parseInt(upperSql.substring(afterLimit, endIndex).trim());
                if (limit > MAX_PAGE_SIZE) {
                    String newSql = sql.substring(0, limitIndex) + "LIMIT " + MAX_PAGE_SIZE + 
                            sql.substring(limitIndex + 5 + String.valueOf(limit).length());
                    com.houyu.common.app.util.SqlUtils.setSql(statementHandler, newSql);
                }
            } catch (NumberFormatException e) {
                // Ignore
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

#### 4.5.4 SqlLogInterceptor 实现

```java
package com.houyu.common.app.mybatis;

import com.houyu.common.app.context.RequestContextHolder;
import com.houyu.common.log.model.HyLogEvent;
import com.houyu.common.log.output.LogOutputManager;
import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.plugin.*;
import org.apache.ibatis.session.ResultHandler;
import org.apache.ibatis.session.RowBounds;

import java.time.LocalDateTime;
import java.util.Properties;

@Intercepts({@Signature(
        type = Executor.class,
        method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}),
        @Signature(
        type = Executor.class,
        method = "update",
        args = {MappedStatement.class, Object.class})})
public class SqlLogInterceptor implements Interceptor {

    private final LogOutputManager logOutputManager;

    public SqlLogInterceptor(LogOutputManager logOutputManager) {
        this.logOutputManager = logOutputManager;
    }

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        long startTime = System.currentTimeMillis();
        
        MappedStatement mappedStatement = (MappedStatement) invocation.getArgs()[0];
        Object parameter = invocation.getArgs()[1];
        BoundSql boundSql = mappedStatement.getBoundSql(parameter);
        
        String sql = boundSql.getSql();
        Object[] parameters = boundSql.getParameterObject() != null ? 
                new Object[]{boundSql.getParameterObject()} : null;

        Object result = null;
        Exception exception = null;
        
        try {
            result = invocation.proceed();
            return result;
        } catch (Exception e) {
            exception = e;
            throw e;
        } finally {
            long executionTime = System.currentTimeMillis() - startTime;
            
            HyLogEvent logEvent = new HyLogEvent();
            logEvent.setTraceId(RequestContextHolder.getTraceId());
            logEvent.setTimestamp(LocalDateTime.now());
            logEvent.setLevel(exception != null ? 
                    com.houyu.common.log.model.LogLevel.ERROR : 
                    com.houyu.common.log.model.LogLevel.DEBUG);
            logEvent.setMessage("SQL Execution");
            logEvent.setServiceName("hy-common-app");
            logEvent.setMethodName(mappedStatement.getId());
            logEvent.setExecutionTime(executionTime);
            logEvent.setSuccess(exception == null);
            
            if (parameters != null) {
                logEvent.setMessage("SQL: " + sql + ", Params: " + parameters.toString());
            } else {
                logEvent.setMessage("SQL: " + sql);
            }

            if (exception != null) {
                logEvent.setExceptionClassName(exception.getClass().getName());
                logEvent.setExceptionMessage(exception.getMessage());
            }

            logOutputManager.output(logEvent);
        }
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

### 4.6 公共 Entity

#### 4.6.1 BaseEntity 实现

```java
package com.houyu.common.app.entity;

import com.fasterxml.jackson.annotation.JsonFormat;
import io.swagger.v3.oas.annotations.media.Schema;
import lombok.Data;

import java.time.LocalDateTime;

@Data
@Schema(description = "公共实体基类")
public class BaseEntity {

    @Schema(description = "自增主键（内部排序，禁止插队）")
    private Long id;

    @Schema(description = "是否当前有效版本")
    private Boolean isCurrent;

    @Schema(description = "日志链路追踪ID")
    private String traceId;

    @Schema(description = "数据拥有人")
    private String owner;

    @Schema(description = "创建人")
    private String creater;

    @Schema(description = "创建时间")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime createTime;

    @Schema(description = "修改人")
    private String updater;

    @Schema(description = "修改时间")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime updateTime;

    @Schema(description = "操作类型（insert,update,delete）")
    private String opType;

    @Schema(description = "备注")
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