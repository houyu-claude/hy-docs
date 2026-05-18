# hy-gateway 架构设计文档

## 1. 模块概述

### 1.1 模块定位

- **模块名称**：hy-gateway
- **父模块**：hy-common-log
- **模块类型**：后端 API 网关模块
- **核心职责**：作为前端请求的统一入口，负责请求路由、安全认证、流量控制和日志记录

### 1.2 设计目标

- 统一入口管理：所有前端请求通过网关进入后端系统
- 链路追踪：生成和传递 traceId，支持全链路追踪
- 安全防护：提供 token 验证、权限校验、安全过滤等能力
- 流量控制：支持限流、熔断等保护机制
- 统一日志：记录网关层请求日志，支持问题排查和审计
- 动态路由：支持基于 Nacos 配置的动态路由规则

### 1.3 功能范围

| 功能 | 说明 | 状态 |
| ---- | ---- | ---- |
| 链路追踪 | 生成并传递 traceId，集成 hy-common-log | 必须 |
| Token 验证 | JWT/OAuth2 token 解析和验证 | 必须 |
| 权限验证 | 基于角色/资源的访问控制 | 必须 |
| 限流控制 | 基于令牌桶/漏桶算法的限流 | 必须 |
| 安全验证 | XSS、SQL 注入、请求参数校验 | 必须 |
| 日志记录 | 网关访问日志记录 | 必须 |
| 路由转发 | 请求路由至后端服务 | 必须 |
| 熔断降级 | 服务熔断和降级处理 | 建议 |
| 负载均衡 | 基于 Spring Cloud LoadBalancer 的负载均衡 | 建议 |

## 2. 架构设计

### 2.1 整体架构图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              前端客户端                                       │
│    ┌──────────┐   ┌──────────┐   ┌──────────┐                              │
│    │  Browser │   │  Mobile  │   │  API SDK │                              │
│    └────┬─────┘   └────┬─────┘   └────┬─────┘                              │
└─────────┼──────────────┼──────────────┼─────────────────────────────────────┘
          │              │              │
          │              │              │ HTTP/HTTPS
          ▼              ▼              ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           hy-gateway                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                     Gateway Filter 链                                │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │  TraceFilter │→│ TokenFilter  │→│ AuthFilter   │→              │   │
│  │  │  生成traceId │  │  Token验证   │  │  权限校验    │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  │         ↓                                                          │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │ RateLimitFilter││SecurityFilter││ RequestLogFilter│             │   │
│  │  │   限流控制    │  │ 安全验证     │  │  日志记录    │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  └────────────────────────────────┬───────────────────────────────────┘   │
│                                   │                                        │
│                                   ▼                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      Route Locator                                  │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  DynamicRouteLocator (基于 Nacos 配置动态路由)                │    │   │
│  │  │  - 路由规则配置                                                │    │   │
│  │  │  - 服务发现 (Nacos Discovery)                                 │    │   │
│  │  │  - 负载均衡 (Spring Cloud LoadBalancer)                        │    │   │
│  │  └────────────────────────────────┬──────────────────────────────┘    │   │
│  └───────────────────────────────────┼───────────────────────────────────┘   │
└──────────────────────────────────────┼───────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           后端服务集群                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐                    │
│  │ hy-user  │  │hy-order  │  │ hy-goods │  │   ...    │                    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘                    │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件分层

| 层级 | 组件名称 | 职责说明 |
| ---- | -------- | -------- |
| 接入层 | GatewayController | Spring Cloud Gateway 配置入口 |
| 过滤器层 | TraceFilter | 生成和传递 traceId |
| 过滤器层 | TokenFilter | JWT Token 解析和验证 |
| 过滤器层 | AuthFilter | 基于 RBAC 的权限校验 |
| 过滤器层 | RateLimitFilter | 限流控制（令牌桶算法） |
| 过滤器层 | SecurityFilter | XSS/SQL 注入防护 |
| 过滤器层 | RequestLogFilter | 请求日志记录 |
| 路由层 | DynamicRouteLocator | 基于 Nacos 的动态路由 |
| 服务发现层 | NacosDiscoveryClient | 服务注册与发现 |
| 负载均衡层 | SpringCloudLoadBalancer | 客户端负载均衡 |
| 日志层 | HyGatewayLogService | 网关日志服务（集成 hy-common-log） |
| 配置层 | GatewayProperties | 网关配置属性 |
| 限流层 | RateLimitManager | 限流管理器（基于 JetCache） |
| 配置层 | WhitelistProperties | 白名单配置属性 |

### 2.3 数据流

```
请求入口
    ↓
TraceFilter (生成/传递 traceId)
    ↓
TokenFilter (解析验证 Token)
    ↓
AuthFilter (权限校验)
    ↓
RateLimitFilter (限流检查)
    ↓
SecurityFilter (安全验证)
    ↓
RequestLogFilter (记录请求日志)
    ↓
RouteLocator (路由匹配)
    ↓
LoadBalancer (负载均衡)
    ↓
后端服务调用
    ↓
ResponseLogFilter (记录响应日志)
    ↓
响应返回
```

## 3. 模块结构

### 3.1 目录结构

```
hy-gateway/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/houyu/gateway/
│   │   │       ├── config/
│   │   │       │   ├── GatewayAutoConfiguration.java
│   │   │       │   ├── GatewayProperties.java
│   │   │       │   ├── NacosConfigProperties.java
│   │   │       │   ├── WhitelistProperties.java
│   │   │       │   └── CorsConfig.java
│   │   │       ├── filter/
│   │   │       │   ├── TraceFilter.java
│   │   │       │   ├── TokenFilter.java
│   │   │       │   ├── AuthFilter.java
│   │   │       │   ├── RateLimitFilter.java
│   │   │       │   ├── SecurityFilter.java
│   │   │       │   ├── RequestLogFilter.java
│   │   │       │   └── ResponseLogFilter.java
│   │   │       ├── route/
│   │   │       │   ├── DynamicRouteLocator.java
│   │   │       │   └── RouteConfigLoader.java
│   │   │       ├── service/
│   │   │       │   ├── TokenService.java
│   │   │       │   ├── AuthService.java
│   │   │       │   ├── RateLimitService.java
│   │   │       │   ├── GatewayLogService.java
│   │   │       │   └── PermissionClient.java
│   │   │       ├── model/
│   │   │       │   ├── GatewayLogEvent.java
│   │   │       │   ├── RouteRule.java
│   │   │       │   └── RateLimitConfig.java
│   │   │       ├── util/
│   │   │       │   ├── JwtUtils.java
│   │   │       │   └── IpUtils.java
│   │   │       └── exception/
│   │   │           ├── GatewayException.java
│   │   │           └── GatewayExceptionHandler.java
│   │   └── resources/
│   │       ├── META-INF/
│   │       │   └── spring/
│   │       │       └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
│   │       └── bootstrap.yml
│   └── test/
│       └── java/
│           └── com/houyu/gateway/
│               ├── filter/
│               │   ├── TraceFilterTest.java
│               │   └── TokenFilterTest.java
│               └── service/
│                   └── RateLimitServiceTest.java
└── pom.xml
```

### 3.2 关键类职责

| 类名 | 职责 | 所属包 |
| ---- | ---- | ------ |
| TraceFilter | 生成 traceId 并放入 MDC | filter |
| TokenFilter | 解析 Authorization Header 中的 JWT Token | filter |
| AuthFilter | 根据 Token 中的用户信息进行权限校验 | filter |
| RateLimitFilter | 基于令牌桶算法进行限流控制 | filter |
| SecurityFilter | 对请求参数进行安全校验（XSS/SQL注入） | filter |
| RequestLogFilter | 记录请求日志到 hy-common-log | filter |
| DynamicRouteLocator | 从 Nacos 配置中心加载动态路由规则 | route |
| TokenService | Token 解析和验证逻辑 | service |
| AuthService | 权限校验逻辑，支持 JetCache + PermissionClient 加载权限数据 | service |
| RateLimitService | 限流管理逻辑（基于 JetCache） | service |
| GatewayLogService | 网关日志服务 | service |
| PermissionClient | 权限数据远程调用客户端 | service |

## 4. 核心功能设计

### 4.1 链路追踪（TraceId 生成）

#### 4.1.1 设计说明

网关作为请求入口，负责生成链路追踪 ID（traceId），并将其传递给下游服务。集成 `hy-common-log` 模块，复用其 `TraceIdGenerator`。

#### 4.1.2 TraceFilter 实现

```java
package com.houyu.gateway.filter;

import com.houyu.common.log.trace.TraceIdGenerator;
import com.houyu.common.log.trace.TraceContextHolder;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpHeaders;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.HashMap;
import java.util.Map;

@Component
public class TraceFilter implements GlobalFilter, Ordered {

    private static final String TRACE_ID_HEADER = "X-Trace-Id";
    private static final String SPAN_ID_HEADER = "X-Span-Id";
    private static final String TRACE_FLAG_HEADER = "X-Trace-Flag";

    private final TraceIdGenerator traceIdGenerator;

    public TraceFilter(TraceIdGenerator traceIdGenerator) {
        this.traceIdGenerator = traceIdGenerator;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String incomingTraceId = exchange.getRequest().getHeaders().getFirst(TRACE_ID_HEADER);
        String incomingSpanId = exchange.getRequest().getHeaders().getFirst(SPAN_ID_HEADER);
        String traceFlag = exchange.getRequest().getHeaders().getFirst(TRACE_FLAG_HEADER);

        String traceId = (incomingTraceId != null && !incomingTraceId.isEmpty())
                ? incomingTraceId
                : traceIdGenerator.generateTraceId(traceFlag);
        
        String spanId = traceIdGenerator.generateSpanId();

        TraceContextHolder.setTraceId(traceId);
        TraceContextHolder.setSpanId(spanId);
        TraceContextHolder.setParentSpanId(incomingSpanId);

        ServerWebExchange mutatedExchange = exchange.mutate()
                .request(r -> r.headers(headers -> {
                    headers.set(TRACE_ID_HEADER, traceId);
                    headers.set(SPAN_ID_HEADER, spanId);
                }))
                .build();

        return chain.filter(mutatedExchange).doFinally(signalType -> {
            TraceContextHolder.clear();
        });
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
}
```

### 4.2 Token 验证

#### 4.2.1 设计说明

网关负责验证前端传入的 JWT Token，解析用户信息并传递给下游服务。Token 从 `Authorization` Header 获取，格式为 `Bearer <token>`。

#### 4.2.2 TokenService 实现

```java
package com.houyu.gateway.service;

import com.houyu.gateway.exception.GatewayException;
import com.houyu.gateway.util.JwtUtils;
import io.jsonwebtoken.Claims;
import org.springframework.stereotype.Service;

import java.util.Map;

@Service
public class TokenService {

    private final JwtUtils jwtUtils;

    public TokenService(JwtUtils jwtUtils) {
        this.jwtUtils = jwtUtils;
    }

    public Map<String, Object> validateAndParseToken(String token) {
        if (token == null || token.isEmpty()) {
            throw new GatewayException("TOKEN_MISSING", "Token is missing");
        }

        String jwtToken = extractJwtToken(token);
        
        try {
            Claims claims = jwtUtils.parseToken(jwtToken);
            return jwtUtils.convertClaimsToMap(claims);
        } catch (Exception e) {
            throw new GatewayException("TOKEN_INVALID", "Invalid token: " + e.getMessage());
        }
    }

    private String extractJwtToken(String token) {
        if (token.startsWith("Bearer ")) {
            return token.substring(7);
        }
        return token;
    }

    public String getUserIdFromToken(String token) {
        Map<String, Object> claims = validateAndParseToken(token);
        return claims.getOrDefault("userId", "").toString();
    }

    public String getUsernameFromToken(String token) {
        Map<String, Object> claims = validateAndParseToken(token);
        return claims.getOrDefault("username", "").toString();
    }

    public String getTenantIdFromToken(String token) {
        Map<String, Object> claims = validateAndParseToken(token);
        return claims.getOrDefault("tenantId", "").toString();
    }
}
```

#### 4.2.3 TokenFilter 实现

```java
package com.houyu.gateway.filter;

import com.houyu.gateway.config.WhitelistProperties;
import com.houyu.gateway.service.TokenService;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.List;

@Component
public class TokenFilter implements GlobalFilter, Ordered {

    private static final String AUTHORIZATION_HEADER = "Authorization";

    private final TokenService tokenService;
    private final WhitelistProperties whitelistProperties;

    public TokenFilter(TokenService tokenService, WhitelistProperties whitelistProperties) {
        this.tokenService = tokenService;
        this.whitelistProperties = whitelistProperties;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().value();
        
        if (isWhitelisted(path)) {
            return chain.filter(exchange);
        }

        String token = exchange.getRequest().getHeaders().getFirst(AUTHORIZATION_HEADER);
        
        if (token == null || token.isEmpty()) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        try {
            String userId = tokenService.getUserIdFromToken(token);
            String username = tokenService.getUsernameFromToken(token);
            String tenantId = tokenService.getTenantIdFromToken(token);

            ServerWebExchange mutatedExchange = exchange.mutate()
                    .request(r -> r.headers(headers -> {
                        headers.set("X-User-Id", userId);
                        headers.set("X-Username", username);
                        headers.set("X-Tenant-Id", tenantId);
                    }))
                    .build();

            return chain.filter(mutatedExchange);
        } catch (Exception e) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
    }

    private boolean isWhitelisted(String path) {
        List<String> whitelist = whitelistProperties.getPaths();
        if (whitelist == null || whitelist.isEmpty()) {
            return false;
        }
        return whitelist.stream().anyMatch(path::startsWith);
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE + 1;
    }
}
```

### 4.3 权限验证

#### 4.3.1 设计说明

基于 RBAC（角色-权限-资源）模型进行权限验证。网关从 Token 中获取用户信息，然后校验用户是否有权限访问当前资源。

#### 4.3.2 AuthService 实现

```java
package com.houyu.gateway.service;

import com.alicp.jetcache.Cache;
import com.alicp.jetcache.anno.CacheType;
import com.alicp.jetcache.anno.CreateCache;
import com.houyu.gateway.exception.GatewayException;
import org.springframework.stereotype.Service;

import java.util.Set;
import java.util.concurrent.TimeUnit;

@Service
public class AuthService {

    @CreateCache(name = "gateway:permissions:",
                 cacheType = CacheType.REMOTE,
                 expire = 300,
                 timeUnit = TimeUnit.SECONDS)
    private Cache<String, Set<String>> permissionCache;

    private final PermissionClient permissionClient;

    public AuthService(PermissionClient permissionClient) {
        this.permissionClient = permissionClient;
    }

    public void checkPermission(String userId, String uri, String method) {
        if (userId == null || userId.isEmpty()) {
            throw new GatewayException("UNAUTHORIZED", "User not authenticated");
        }

        Set<String> permissions = getPermissions(userId);
        
        String requiredPermission = buildPermissionKey(uri, method);
        
        if (!permissions.contains(requiredPermission)) {
            throw new GatewayException("FORBIDDEN", "Insufficient permissions");
        }
    }

    private Set<String> getPermissions(String userId) {
        return permissionCache.computeIfAbsent(userId, this::loadPermissionsFromRemote);
    }

    private Set<String> loadPermissionsFromRemote(String userId) {
        return permissionClient.fetchUserPermissions(userId);
    }

    private String buildPermissionKey(String uri, String method) {
        return method.toUpperCase() + ":" + uri;
    }

    public void refreshUserPermissions(String userId) {
        permissionCache.remove(userId);
    }
}
```

#### 4.3.3 AuthFilter 实现

```java
package com.houyu.gateway.filter;

import com.houyu.gateway.service.AuthService;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Component
public class AuthFilter implements GlobalFilter, Ordered {

    private static final String USER_ID_HEADER = "X-User-Id";

    private final AuthService authService;

    public AuthFilter(AuthService authService) {
        this.authService = authService;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String userId = exchange.getRequest().getHeaders().getFirst(USER_ID_HEADER);
        String uri = exchange.getRequest().getPath().value();
        String method = exchange.getRequest().getMethod().name();

        try {
            authService.checkPermission(userId, uri, method);
        } catch (Exception e) {
            exchange.getResponse().setStatusCode(org.springframework.http.HttpStatus.FORBIDDEN);
            return exchange.getResponse().setComplete();
        }

        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE + 2;
    }
}
```

### 4.4 限流控制

#### 4.4.1 设计说明

基于令牌桶算法实现限流，使用 JetCache 存储限流状态，支持多种限流维度：
- IP 限流
- 用户限流
- 接口限流

#### 4.4.2 RateLimitService 实现

```java
package com.houyu.gateway.service;

import com.alicp.jetcache.Cache;
import com.alicp.jetcache.anno.CacheType;
import com.alicp.jetcache.anno.CreateCache;
import com.houyu.gateway.exception.GatewayException;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

@Service
public class RateLimitService {

    @CreateCache(name = "gateway:ratelimit:",
                 cacheType = CacheType.REMOTE,
                 expire = 60,
                 timeUnit = TimeUnit.SECONDS)
    private Cache<String, AtomicInteger> rateLimitCache;

    public boolean tryAcquire(String key, int limit, int windowSeconds) {
        AtomicInteger counter = rateLimitCache.computeIfAbsent(key, k -> new AtomicInteger(0));
        
        int current = counter.incrementAndGet();
        
        if (current == 1) {
            rateLimitCache.expire(key, windowSeconds, TimeUnit.SECONDS);
        }
        
        if (current > limit) {
            return false;
        }
        
        return true;
    }

    public void checkRateLimit(String clientIp, String userId, String uri) {
        String ipKey = "ip:" + clientIp;
        String userKey = userId != null ? "user:" + userId : null;
        String uriKey = "uri:" + uri;

        if (!tryAcquire(ipKey, 100, 60)) {
            throw new GatewayException("RATE_LIMIT_IP", "IP rate limit exceeded");
        }

        if (userKey != null && !tryAcquire(userKey, 50, 60)) {
            throw new GatewayException("RATE_LIMIT_USER", "User rate limit exceeded");
        }

        if (!tryAcquire(uriKey, 500, 60)) {
            throw new GatewayException("RATE_LIMIT_URI", "URI rate limit exceeded");
        }
    }
}
```

#### 4.4.3 RateLimitFilter 实现

```java
package com.houyu.gateway.filter;

import com.houyu.gateway.service.RateLimitService;
import com.houyu.gateway.util.IpUtils;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Component
public class RateLimitFilter implements GlobalFilter, Ordered {

    private static final String USER_ID_HEADER = "X-User-Id";

    private final RateLimitService rateLimitService;

    public RateLimitFilter(RateLimitService rateLimitService) {
        this.rateLimitService = rateLimitService;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String clientIp = IpUtils.getClientIp(exchange.getRequest());
        String userId = exchange.getRequest().getHeaders().getFirst(USER_ID_HEADER);
        String uri = exchange.getRequest().getPath().value();

        try {
            rateLimitService.checkRateLimit(clientIp, userId, uri);
        } catch (Exception e) {
            exchange.getResponse().setStatusCode(org.springframework.http.HttpStatus.TOO_MANY_REQUESTS);
            return exchange.getResponse().setComplete();
        }

        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE + 3;
    }
}
```

### 4.5 安全验证

#### 4.5.1 设计说明

对请求参数进行安全校验，防止 XSS 攻击和 SQL 注入。

#### 4.5.2 SecurityFilter 实现

```java
package com.houyu.gateway.filter;

import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.regex.Pattern;

@Component
public class SecurityFilter implements GlobalFilter, Ordered {

    private static final Pattern XSS_PATTERN = Pattern.compile(
            "<script[^>]*>.*?</script>|javascript:|onload=|onclick=|onerror=|eval\\(",
            Pattern.CASE_INSENSITIVE
    );

    private static final Pattern SQL_INJECTION_PATTERN = Pattern.compile(
            "(?i)(select|insert|update|delete|drop|union|exec|execute|xp_)",
            Pattern.CASE_INSENSITIVE
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String queryString = exchange.getRequest().getQueryParams().toString();
        String path = exchange.getRequest().getPath().value();

        if (containsXss(queryString) || containsXss(path)) {
            exchange.getResponse().setStatusCode(org.springframework.http.HttpStatus.BAD_REQUEST);
            return exchange.getResponse().setComplete();
        }

        if (containsSqlInjection(queryString) || containsSqlInjection(path)) {
            exchange.getResponse().setStatusCode(org.springframework.http.HttpStatus.BAD_REQUEST);
            return exchange.getResponse().setComplete();
        }

        return chain.filter(exchange);
    }

    private boolean containsXss(String input) {
        return input != null && XSS_PATTERN.matcher(input).find();
    }

    private boolean containsSqlInjection(String input) {
        return input != null && SQL_INJECTION_PATTERN.matcher(input).find();
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE + 4;
    }
}
```

### 4.6 日志记录

#### 4.6.1 设计说明

网关日志服务集成 `hy-common-log` 模块，记录请求和响应日志。

#### 4.6.2 GatewayLogService 实现

```java
package com.houyu.gateway.service;

import com.houyu.common.log.model.HyLogEvent;
import com.houyu.common.log.model.HttpRequestInfo;
import com.houyu.common.log.model.HttpResponseInfo;
import com.houyu.common.log.output.LogOutputManager;
import com.houyu.common.log.trace.TraceContextHolder;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@Service
public class GatewayLogService {

    private final LogOutputManager logOutputManager;

    public GatewayLogService(LogOutputManager logOutputManager) {
        this.logOutputManager = logOutputManager;
    }

    public void logRequest(String clientIp, String method, String uri, String queryString,
                          Map<String, String> headers, String requestBody) {
        HyLogEvent logEvent = createLogEvent();
        logEvent.setMessage("Gateway incoming request");
        
        HttpRequestInfo httpRequest = new HttpRequestInfo();
        httpRequest.setMethod(method);
        httpRequest.setUri(uri);
        httpRequest.setQueryString(queryString);
        httpRequest.setHeaders(headers);
        httpRequest.setRequestBody(requestBody);
        httpRequest.setClientIp(clientIp);
        
        logEvent.setHttpRequest(httpRequest);
        
        logOutputManager.output(logEvent);
    }

    public void logResponse(int statusCode, String responseBody, long executionTime) {
        HyLogEvent logEvent = createLogEvent();
        logEvent.setMessage("Gateway response");
        logEvent.setExecutionTime(executionTime);
        logEvent.setSuccess(statusCode >= 200 && statusCode < 400);
        
        HttpResponseInfo httpResponse = new HttpResponseInfo();
        httpResponse.setStatusCode(statusCode);
        httpResponse.setResponseBody(responseBody);
        
        logEvent.setHttpResponse(httpResponse);
        
        logOutputManager.output(logEvent);
    }

    private HyLogEvent createLogEvent() {
        HyLogEvent logEvent = new HyLogEvent();
        logEvent.setTraceId(TraceContextHolder.getTraceId());
        logEvent.setSpanId(TraceContextHolder.getSpanId());
        logEvent.setParentSpanId(TraceContextHolder.getParentSpanId());
        logEvent.setTimestamp(LocalDateTime.now());
        logEvent.setServiceName("hy-gateway");
        logEvent.setMdcContext(new HashMap<>(TraceContextHolder.getMdcContext()));
        
        return logEvent;
    }
}
```

#### 4.6.3 RequestLogFilter 实现

```java
package com.houyu.gateway.filter;

import com.houyu.gateway.service.GatewayLogService;
import com.houyu.gateway.util.IpUtils;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.HashMap;
import java.util.Map;

@Component
public class RequestLogFilter implements GlobalFilter, Ordered {

    private final GatewayLogService gatewayLogService;

    public RequestLogFilter(GatewayLogService gatewayLogService) {
        this.gatewayLogService = gatewayLogService;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String clientIp = IpUtils.getClientIp(exchange.getRequest());
        String method = exchange.getRequest().getMethod().name();
        String uri = exchange.getRequest().getPath().value();
        String queryString = exchange.getRequest().getQueryParams().toString();
        
        Map<String, String> headers = new HashMap<>();
        exchange.getRequest().getHeaders().forEach((key, value) -> {
            headers.put(key, value.get(0));
        });

        gatewayLogService.logRequest(clientIp, method, uri, queryString, headers, null);

        long startTime = System.currentTimeMillis();

        return chain.filter(exchange).doFinally(signalType -> {
            long executionTime = System.currentTimeMillis() - startTime;
            int statusCode = exchange.getResponse().getStatusCode() != null 
                    ? exchange.getResponse().getStatusCode().value() 
                    : 500;
            gatewayLogService.logResponse(statusCode, null, executionTime);
        });
    }

    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;
    }
}
```

### 4.7 动态路由

#### 4.7.1 设计说明

基于 Nacos 配置中心实现动态路由，支持运行时修改路由规则无需重启服务。

#### 4.7.2 DynamicRouteLocator 实现

```java
package com.houyu.gateway.route;

import com.alibaba.nacos.api.config.ConfigService;
import com.alibaba.nacos.api.config.listener.Listener;
import com.alibaba.nacos.api.exception.NacosException;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.houyu.gateway.model.RouteRule;
import org.springframework.cloud.gateway.route.RouteDefinition;
import org.springframework.cloud.gateway.route.RouteDefinitionLocator;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Flux;

import jakarta.annotation.PostConstruct;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

@Component
public class DynamicRouteLocator implements RouteDefinitionLocator {

    private static final String NACOS_DATA_ID = "gateway-routes";
    private static final String NACOS_GROUP = "DEFAULT_GROUP";

    private final ConfigService configService;
    private final ObjectMapper objectMapper;
    private final List<RouteDefinition> routeDefinitions = new CopyOnWriteArrayList<>();

    public DynamicRouteLocator(ConfigService configService, ObjectMapper objectMapper) {
        this.configService = configService;
        this.objectMapper = objectMapper;
    }

    @PostConstruct
    public void init() throws NacosException {
        loadRoutes();
        addConfigListener();
    }

    private void loadRoutes() throws NacosException {
        String config = configService.getConfig(NACOS_DATA_ID, NACOS_GROUP, 3000L);
        if (config != null && !config.isEmpty()) {
            List<RouteRule> rules = objectMapper.readValue(config, 
                    new TypeReference<List<RouteRule>>() {});
            routeDefinitions.clear();
            rules.forEach(rule -> routeDefinitions.add(rule.toRouteDefinition()));
        }
    }

    private void addConfigListener() {
        try {
            configService.addListener(NACOS_DATA_ID, NACOS_GROUP, new Listener() {
                @Override
                public void receiveConfigInfo(String config) {
                    try {
                        List<RouteRule> rules = objectMapper.readValue(config,
                                new TypeReference<List<RouteRule>>() {});
                        routeDefinitions.clear();
                        rules.forEach(rule -> routeDefinitions.add(rule.toRouteDefinition()));
                    } catch (Exception e) {
                        // log error
                    }
                }

                @Override
                public Executor getExecutor() {
                    return null;
                }
            });
        } catch (NacosException e) {
            // log error
        }
    }

    @Override
    public Flux<RouteDefinition> getRouteDefinitions() {
        return Flux.fromIterable(routeDefinitions);
    }
}
```

#### 4.7.3 RouteRule 模型

```java
package com.houyu.gateway.model;

import lombok.Data;
import org.springframework.cloud.gateway.route.RouteDefinition;
import org.springframework.cloud.gateway.route.RouteDefinitionLocator;

import java.util.List;

@Data
public class RouteRule {
    
    private String id;
    private String uri;
    private List<PredicateDefinition> predicates;
    private List<FilterDefinition> filters;
    private Integer order;

    @Data
    public static class PredicateDefinition {
        private String name;
        private String value;
    }

    @Data
    public static class FilterDefinition {
        private String name;
        private String value;
    }

    public RouteDefinition toRouteDefinition() {
        RouteDefinition definition = new RouteDefinition();
        definition.setId(this.id);
        definition.setUri(java.net.URI.create(this.uri));
        definition.setOrder(this.order != null ? this.order : 0);
        
        this.predicates.forEach(predicate -> {
            org.springframework.cloud.gateway.route.PredicateDefinition pd = 
                    new org.springframework.cloud.gateway.route.PredicateDefinition();
            pd.setName(predicate.getName());
            pd.addArg("_genkey_0", predicate.getValue());
            definition.getPredicates().add(pd);
        });
        
        this.filters.forEach(filter -> {
            org.springframework.cloud.gateway.route.FilterDefinition fd = 
                    new org.springframework.cloud.gateway.route.FilterDefinition();
            fd.setName(filter.getName());
            fd.addArg("_genkey_0", filter.getValue());
            definition.getFilters().add(fd);
        });
        
        return definition;
    }
}
```

### 4.8 白名单路由机制

#### 4.8.1 设计说明

支持配置无需 Token 验证的白名单路径，用于开放接口如登录、健康检查等。白名单配置通过 `WhitelistProperties` 管理，并与 `TokenFilter` 联动。

#### 4.8.2 WhitelistProperties 实现

```java
package com.houyu.gateway.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.util.ArrayList;
import java.util.List;

@Data
@Component
@ConfigurationProperties(prefix = "hy.gateway.whitelist")
public class WhitelistProperties {

    private List<String> paths = new ArrayList<>();
}
```

### 4.9 CORS 跨域配置

#### 4.9.1 设计说明

通过 `CorsWebFilter` 实现全局跨域配置，支持配置允许的源、方法、请求头和凭证。

#### 4.9.2 CorsConfig 实现

```java
package com.houyu.gateway.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.reactive.CorsWebFilter;
import org.springframework.web.cors.reactive.UrlBasedCorsConfigurationSource;

import java.util.Arrays;
import java.util.List;

@Configuration
public class CorsConfig {

    @Bean
    public CorsWebFilter corsWebFilter() {
        CorsConfiguration config = new CorsConfiguration();
        
        config.setAllowedOriginPatterns(List.of("http://localhost:3000", "https://*.houyu.com"));
        config.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);

        return new CorsWebFilter(source);
    }
}
```

### 4.10 PermissionClient 权限数据加载

#### 4.10.1 设计说明

通过 `PermissionClient` 从权限服务获取用户权限数据，配合 `JetCache` 缓存权限信息，减少远程调用。

#### 4.10.2 PermissionClient 实现

```java
package com.houyu.gateway.service;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;

import java.util.Set;

@FeignClient(name = "hy-auth")
public interface PermissionClient {

    @GetMapping("/api/permission/user")
    Set<String> fetchUserPermissions(@RequestParam("userId") String userId);
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
        <artifactId>hy-common-log</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>

    <groupId>com.houyu</groupId>
    <artifactId>hy-gateway</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>
    <name>hy-gateway</name>
    <description>HouYu Gateway Module - API Gateway with Spring Cloud Alibaba</description>

    <dependencies>
        <!-- Spring Boot Starter WebFlux -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webflux</artifactId>
        </dependency>

        <!-- Spring Cloud Gateway -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>

        <!-- Spring Cloud Alibaba Nacos Discovery -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>

        <!-- Spring Cloud Alibaba Nacos Config -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
        </dependency>

        <!-- Spring Cloud LoadBalancer -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-loadbalancer</artifactId>
        </dependency>

        <!-- JetCache -->
        <dependency>
            <groupId>com.alicp.jetcache</groupId>
            <artifactId>jetcache-starter-redis-lettuce</artifactId>
        </dependency>

        <!-- JWT -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <scope>runtime</scope>
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

### 6.1 bootstrap.yml

```yaml
spring:
  application:
    name: hy-gateway
  cloud:
    nacos:
      discovery:
        server-addr: ${NACOS_SERVER_ADDR:localhost:8848}
        namespace: ${NACOS_NAMESPACE:public}
      config:
        server-addr: ${NACOS_SERVER_ADDR:localhost:8848}
        namespace: ${NACOS_NAMESPACE:public}
        group: DEFAULT_GROUP
        file-extension: yaml
        extension-configs:
          - data-id: gateway-routes
            group: DEFAULT_GROUP
            refresh: true

server:
  port: 8080

hy:
  log:
    enabled: true
  gateway:
    rate-limit:
      ip-limit: 100
      user-limit: 50
      uri-limit: 500
    whitelist:
      paths:
        - /api/public/**
        - /api/auth/login
        - /health

spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            allowed-origins:
              - http://localhost:3000
              - https://*.houyu.com
            allowed-methods:
              - GET
              - POST
              - PUT
              - DELETE
              - OPTIONS
            allowed-headers:
              - *
            allow-credentials: true
            max-age: 3600
```

### 6.2 Nacos 路由配置示例 (gateway-routes)

```json
[
  {
    "id": "hy-user-route",
    "uri": "lb://hy-user",
    "order": 1,
    "predicates": [
      {"name": "Path", "value": "/api/user/**"}
    ],
    "filters": [
      {"name": "StripPrefix", "value": "1"}
    ]
  },
  {
    "id": "hy-order-route",
    "uri": "lb://hy-order",
    "order": 2,
    "predicates": [
      {"name": "Path", "value": "/api/order/**"}
    ],
    "filters": [
      {"name": "StripPrefix", "value": "1"}
    ]
  }
]
```

## 7. 异常处理

### 7.1 异常类型

| 异常码 | 异常信息 | HTTP状态码 |
| ------ | -------- | ---------- |
| TOKEN_MISSING | Token is missing | 401 |
| TOKEN_INVALID | Invalid token | 401 |
| UNAUTHORIZED | User not authenticated | 401 |
| FORBIDDEN | Insufficient permissions | 403 |
| RATE_LIMIT_IP | IP rate limit exceeded | 429 |
| RATE_LIMIT_USER | User rate limit exceeded | 429 |
| RATE_LIMIT_URI | URI rate limit exceeded | 429 |
| BAD_REQUEST | Bad request | 400 |

### 7.2 GatewayExceptionHandler 实现

```java
package com.houyu.gateway.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import reactor.core.publisher.Mono;

import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GatewayExceptionHandler {

    @ExceptionHandler(GatewayException.class)
    public Mono<ResponseEntity<Map<String, Object>>> handleGatewayException(GatewayException e) {
        Map<String, Object> response = new HashMap<>();
        response.put("code", e.getCode());
        response.put("message", e.getMessage());
        
        HttpStatus status = mapCodeToStatus(e.getCode());
        
        return Mono.just(ResponseEntity.status(status).body(response));
    }

    private HttpStatus mapCodeToStatus(String code) {
        return switch (code) {
            case "TOKEN_MISSING", "TOKEN_INVALID", "UNAUTHORIZED" -> HttpStatus.UNAUTHORIZED;
            case "FORBIDDEN" -> HttpStatus.FORBIDDEN;
            case "RATE_LIMIT_IP", "RATE_LIMIT_USER", "RATE_LIMIT_URI" -> HttpStatus.TOO_MANY_REQUESTS;
            default -> HttpStatus.BAD_REQUEST;
        };
    }
}
```

## 8. 监控指标

| 指标名称 | 类型 | 说明 |
| -------- | ---- | ---- |
| hy.gateway.requests.total | Counter | 总请求数 |
| hy.gateway.requests.success | Counter | 成功请求数 |
| hy.gateway.requests.failure | Counter | 失败请求数 |
| hy.gateway.requests.duration | Histogram | 请求耗时分布 |
| hy.gateway.rate.limit.ip | Counter | IP限流触发次数 |
| hy.gateway.rate.limit.user | Counter | 用户限流触发次数 |
| hy.gateway.rate.limit.uri | Counter | URI限流触发次数 |
| hy.gateway.errors.token | Counter | Token验证失败次数 |
| hy.gateway.errors.auth | Counter | 权限校验失败次数 |

## 9. 安全考虑

1. **敏感信息保护**：日志中不记录完整的 Token 和密码等敏感信息
2. **请求体限制**：限制最大请求体大小，防止 DoS 攻击
3. **IP 白名单**：支持配置 IP 白名单，限制访问来源
4. **HTTPS 强制**：生产环境强制使用 HTTPS
5. **请求签名验证**：支持对关键接口进行请求签名验证
6. **SQL 注入防护**：SecurityFilter 对请求参数进行 SQL 注入检测
7. **XSS 攻击防护**：SecurityFilter 对请求参数进行 XSS 攻击检测