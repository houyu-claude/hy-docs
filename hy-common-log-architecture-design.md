# hy-common-log 架构设计文档

## 1. 模块概述

### 1.1 模块定位
- **模块名称**：hy-common-log
- **父模块**：hy-common
- **模块类型**：后端公共日志模块
- **核心职责**：通过自定义 Logback Appender 接管所有日志输出，进行统一格式化、脱敏、过滤和持久化

### 1.2 设计目标
- 统一日志格式规范（JSON/文本）
- 提供可扩展的日志处理机制
- 支持多种日志输出渠道（控制台、文件、数据库）
- 简化各业务模块的日志配置
- 提供日志脱敏、过滤、链路追踪等增强功能
- 支持 API 重放所需的完整请求响应信息记录

## 2. 架构设计

### 2.1 整体架构图

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              业务应用模块                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │
│  │  hy-user │  │ hy-order │  │ hy-goods│  │  ...     │                  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘                  │
└───────┼─────────────┼─────────────┼─────────────┼──────────────────────────┘
        │             │             │             │
        │  SLF4J/Logback API       │             │
        ▼             ▼             ▼             ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                         Logback 原生框架                                    │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │                    Logger + ILoggingEvent                           │   │
│  └────────────────────────────────────┬───────────────────────────────┘   │
└─────────────────────────────────────────┼──────────────────────────────────┘
                                          │
                                          ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                          hy-common-log                                     │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │                    Logback Appender 接管层                           │   │
│  │  ┌────────────────────────────────────────────────────────────┐   │   │
│  │  │              HyCommonLogAppender (核心)                     │   │   │
│  │  │  - 实现 ch.qos.logback.core.Appender                       │   │   │
│  │  │  - 接管所有 ILoggingEvent 事件                              │   │   │
│  │  │  - 异步处理队列（Disruptor）                                │   │   │
│  │  └──────────────────────┬─────────────────────────────────────┘   │   │
│  └─────────────────────────┼──────────────────────────────────────────┘   │
│                            │                                               │
│                            ▼                                               │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │                    LogEvent 转换层                                   │   │
│  │  ┌────────────────────────────────────────────────────────────┐   │   │
│  │  │              LogEventConverter                              │   │   │
│  │  │  - ILoggingEvent → HyLogEvent 转换                         │   │   │
│  │  │  - 上下文信息补充（MDC、请求信息等）                        │   │   │
│  │  └──────────────────────┬─────────────────────────────────────┘   │   │
│  └─────────────────────────┼──────────────────────────────────────────┘   │
│                            │                                               │
│                            ▼                                               │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │                    日志增强处理层                                    │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │   │
│  │  │   脱敏处理   │  │   过滤规则   │  │   采样策略   │            │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘            │   │
│  └─────────────────────────┬──────────────────────────────────────────┘   │
│                            │                                               │
│                            ▼                                               │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │                    日志格式化层                                      │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │   │
│  │  │  JSON格式化  │  │  文本格式化  │  │  自定义格式  │            │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘            │   │
│  └─────────────────────────┬──────────────────────────────────────────┘   │
│                            │                                               │
│                ┌───────────┴───────────┐                                   │
│                ▼                       ▼                                   │
│  ┌──────────────────────┐  ┌──────────────────────┐                       │
│  │   同步输出通道        │  │   异步输出通道        │                       │
│  │  ┌────────────────┐  │  │  ┌────────────────┐  │                       │
│  │  │  Console输出   │  │  │  │   File输出     │  │                       │
│  │  └────────────────┘  │  │  └────────────────┘  │                       │
│  │                        │  │  ┌────────────────┐  │                       │
│  │                        │  │  │   DB存储       │  │                       │
│  │                        │  │  │  (后置处理器)  │  │                       │
│  │                        │  │  └────────────────┘  │                       │
│  └────────────────────────┘  └──────────────────────┘                       │
└────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件分层

| 层级 | 组件名称 | 职责说明 |
|------|---------|---------|
| 接管层 | HyCommonLogAppender | 实现 ch.qos.logback.core.Appender，接管所有 ILoggingEvent 事件 |
| 转换层 | LogEventConverter | 将 Logback 的 ILoggingEvent 转换为统一的 HyLogEvent 模型 |
| 增强层 | LogDesensitizer | 日志脱敏处理器 |
| 增强层 | LogFilter | 日志过滤器（按级别、关键词等） |
| 增强层 | LogSampler | 日志采样器，降低高并发下的日志量 |
| 格式化层 | LogFormatter | 日志格式化抽象接口 |
| 格式化层 | JsonLogFormatter | JSON 格式日志实现 |
| 格式化层 | TextLogFormatter | 文本格式日志实现 |
| 输出层 | LogAppender | 日志输出器抽象接口 |
| 输出层 | ConsoleLogAppender | 控制台输出实现 |
| 输出层 | FileLogAppender | 文件输出实现 |
| 输出层 | DbLogAppender | 数据库存储实现（后置处理） |
| 上下文层 | TraceIdGenerator | 链路追踪 ID 生成器 |
| 上下文层 | LogContextHolder | 管理日志上下文（MDC），支持链路追踪 |

## 3. 模块结构

### 3.1 目录结构

```
hy-common-log/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/houyu/common/log/
│   │   │       ├── config/              # 配置类
│   │   │       │   ├── LogAutoConfiguration.java
│   │   │       │   └── LogProperties.java
│   │   │       ├── appender/            # Logback Appender 层
│   │   │       │   ├── HyCommonLogAppender.java
│   │   │       │   └── AsyncLogEventQueue.java
│   │   │       ├── converter/           # 转换层
│   │   │       │   └── LogEventConverter.java
│   │   │       ├── model/               # 统一模型
│   │   │       │   ├── HyLogEvent.java
│   │   │       │   ├── HttpRequestInfo.java
│   │   │       │   ├── HttpResponseInfo.java
│   │   │       │   └── LogLevel.java
│   │   │       ├── formatter/           # 格式化层
│   │   │       │   ├── LogFormatter.java
│   │   │       │   ├── JsonLogFormatter.java
│   │   │       │   └── TextLogFormatter.java
│   │   │       ├── desensitizer/        # 脱敏层
│   │   │       │   ├── Desensitizer.java
│   │   │       │   ├── PhoneDesensitizer.java
│   │   │       │   ├── EmailDesensitizer.java
│   │   │       │   ├── IdCardDesensitizer.java
│   │   │       │   ├── BankCardDesensitizer.java
│   │   │       │   ├── PasswordDesensitizer.java
│   │   │       │   └── DesensitizerManager.java
│   │   │       ├── filter/              # 过滤层
│   │   │       │   ├── LogFilter.java
│   │   │       │   ├── LevelLogFilter.java
│   │   │       │   └── KeywordLogFilter.java
│   │   │       ├── sampler/             # 采样层
│   │   │       │   ├── LogSampler.java
│   │   │       │   └── PercentageLogSampler.java
│   │   │       ├── output/              # 输出层
│   │   │       │   ├── LogOutput.java
│   │   │       │   ├── ConsoleLogOutput.java
│   │   │       │   ├── FileLogOutput.java
│   │   │       │   └── DbLogOutput.java
│   │   │       ├── trace/               # 链路追踪
│   │   │       │   ├── TraceIdGenerator.java
│   │   │       │   ├── SnowflakeTraceIdGenerator.java
│   │   │       │   └── TraceContextHolder.java
│   │   │       ├── constant/            # 常量定义
│   │   │       │   └── LogConstants.java
│   │   │       ├── annotation/          # 注解
│   │   │       │   └── LogDesensitize.java
│   │   │       └── util/                # 工具类
│   │   │           └── LogUtils.java
│   │   └── resources/
│   │       ├── META-INF/
│   │       │   └── spring/
│   │       │       └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
│   │       └── sql/
│   │           └── log_table_schema.sql
│   └── test/
│       └── java/
│           └── com/houyu/common/log/
│               └── ...  # 测试类
└── pom.xml
```

### 3.2 Maven 依赖配置

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
    <artifactId>hy-common-log</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>
    <name>hy-common-log</name>
    <description>HouYu Common Log Module - 通过自定义 Logback Appender 实现统一日志管理</description>

    <dependencies>
        <!-- Spring Boot Starter -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <!-- Spring Boot JDBC (用于 DB 存储) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Logback (Spring Boot 默认包含) -->
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
        </dependency>

        <!-- Disruptor 高性能队列 (异步日志处理) -->
        <dependency>
            <groupId>com.lmax</groupId>
            <artifactId>disruptor</artifactId>
            <version>3.4.4</version>
        </dependency>

        <!-- Jackson for JSON formatting -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
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

## 4. 核心功能设计

### 4.1 Logback 接管机制

#### 4.1.1 核心原理

通过 Spring Boot 自动配置类 `LogAutoConfiguration` 编程式注册自定义 Appender，无需手动配置 logback.xml。

#### 4.1.2 HyCommonLogAppender 实现

```java
package com.houyu.common.log.appender;

import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.core.AppenderBase;
import com.lmax.disruptor.RingBuffer;
import com.lmax.disruptor.dsl.Disruptor;
import com.lmax.disruptor.dsl.ProducerType;
import com.lmax.disruptor.BlockingWaitStrategy;

public class HyCommonLogAppender extends AppenderBase<ILoggingEvent> {
    
    private Disruptor<LogEventEvent> disruptor;
    private RingBuffer<LogEventEvent> ringBuffer;
    private final int bufferSize;
    
    public HyCommonLogAppender(int bufferSize) {
        this.bufferSize = bufferSize;
    }
    
    @Override
    public void start() {
        // 初始化 Disruptor 异步队列
        ThreadFactory threadFactory = new ThreadFactoryBuilder()
                .setNameFormat("log-disruptor-%d")
                .build();
        
        disruptor = new Disruptor<>(
                LogEventEvent::new,
                bufferSize,
                threadFactory,
                ProducerType.MULTI,
                new BlockingWaitStrategy()
        );
        
        // 设置事件处理器
        disruptor.handleEventsWith(new LogEventHandler());
        disruptor.start();
        ringBuffer = disruptor.getRingBuffer();
        
        super.start();
    }
    
    @Override
    protected void append(ILoggingEvent eventObject) {
        if (!isStarted()) {
            return;
        }
        
        // 将日志事件发布到 Disruptor 队列
        long sequence = ringBuffer.next();
        try {
            LogEventEvent logEventEvent = ringBuffer.get(sequence);
            logEventEvent.setLoggingEvent(eventObject);
        } finally {
            ringBuffer.publish(sequence);
        }
    }
    
    @Override
    public void stop() {
        super.stop();
        if (disruptor != null) {
            disruptor.shutdown();
        }
    }
}
```

#### 4.1.3 LogEventEvent 事件封装

```java
package com.houyu.common.log.appender;

import ch.qos.logback.classic.spi.ILoggingEvent;
import com.lmax.disruptor.EventFactory;

public class LogEventEvent {
    private ILoggingEvent loggingEvent;
    
    public ILoggingEvent getLoggingEvent() {
        return loggingEvent;
    }
    
    public void setLoggingEvent(ILoggingEvent loggingEvent) {
        this.loggingEvent = loggingEvent;
    }
    
    public static EventFactory<LogEventEvent> getFactory() {
        return LogEventEvent::new;
    }
}
```

#### 4.1.4 LogEventHandler 事件处理器

```java
package com.houyu.common.log.appender;

import com.houyu.common.log.converter.LogEventConverter;
import com.houyu.common.log.model.HyLogEvent;
import com.houyu.common.log.output.LogOutputManager;
import com.lmax.disruptor.EventHandler;

public class LogEventHandler implements EventHandler<LogEventEvent> {
    
    private final LogEventConverter converter = new LogEventConverter();
    private final LogOutputManager outputManager = LogOutputManager.getInstance();
    
    @Override
    public void onEvent(LogEventEvent event, long sequence, boolean endOfBatch) {
        // 转换为统一 HyLogEvent 模型
        HyLogEvent hyLogEvent = converter.convert(event.getLoggingEvent());
        
        // 输出到各个目标（控制台、文件、DB）
        outputManager.output(hyLogEvent);
    }
}
```

#### 4.1.5 LogAutoConfiguration 编程式注册

```java
package com.houyu.common.log.config;

import ch.qos.logback.classic.Logger;
import ch.qos.logback.classic.LoggerContext;
import com.houyu.common.log.appender.HyCommonLogAppender;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Configuration;

import javax.annotation.PostConstruct;

@Configuration
@EnableConfigurationProperties(LogProperties.class)
@ConditionalOnProperty(prefix = "hy.log", name = "enabled", havingValue = "true", matchIfMissing = true)
public class LogAutoConfiguration {
    
    @Autowired
    private LogProperties logProperties;
    
    @PostConstruct
    public void init() {
        // 获取 Logback 上下文
        LoggerContext loggerContext = (LoggerContext) LoggerFactory.getILoggerFactory();
        
        // 创建并配置自定义 Appender
        HyCommonLogAppender appender = new HyCommonLogAppender(
                logProperties.getAppender().getBufferSize()
        );
        appender.setContext(loggerContext);
        appender.setName("HY_COMMON_LOG");
        appender.start();
        
        // 获取根 Logger 并添加 Appender
        Logger rootLogger = loggerContext.getLogger(Logger.ROOT_LOGGER_NAME);
        rootLogger.addAppender(appender);
    }
}
```

### 4.2 统一 LogEvent 模型

#### 4.2.1 HyLogEvent - 完整日志事件模型

```java
package com.houyu.common.log.model;

import lombok.Data;
import java.time.LocalDateTime;
import java.util.Map;

@Data
public class HyLogEvent {
    
    // ========== 基础日志字段 ==========
    private String eventId;                    // 日志事件唯一ID（UUID）
    private LocalDateTime timestamp;           // 日志时间戳
    private LogLevel level;                    // 日志级别
    private String loggerName;                 // Logger 名称
    private String threadName;                 // 线程名称
    private String message;                    // 日志消息
    private String formattedMessage;           // 格式化后的消息
    
    // ========== 异常信息 ==========
    private String exceptionClassName;         // 异常类名
    private String exceptionMessage;           // 异常消息
    private String stackTrace;                 // 异常堆栈
    
    // ========== 调用位置信息 ==========
    private String className;                  // 类名
    private String methodName;                 // 方法名
    private String fileName;                   // 文件名
    private Integer lineNumber;                // 行号
    
    // ========== 链路追踪字段 ==========
    private String traceId;                    // 链路追踪ID
    private String spanId;                     // 当前跨度ID
    private String parentSpanId;               // 父跨度ID
    private String serviceName;                // 服务名称
    private String serviceVersion;             // 服务版本
    private String environment;                // 环境标识（dev/test/prod）
    
    // ========== API 重放支持字段 ==========
    private HttpRequestInfo httpRequest;       // HTTP 请求信息
    private HttpResponseInfo httpResponse;     // HTTP 响应信息
    private Long executionTime;                // 执行耗时（毫秒）
    private Boolean isSuccess;                 // 是否成功
    
    // ========== 用户/租户信息 ==========
    private String userId;                     // 用户ID
    private String username;                   // 用户名
    private String tenantId;                   // 租户ID
    
    // ========== 系统信息 ==========
    private String serverIp;                   // 服务器IP
    private String clientIp;                   // 客户端IP
    private Map<String, String> mdcContext;    // MDC 上下文
    
    // ========== 扩展字段 ==========
    private Map<String, Object> extensions;    // 自定义扩展字段
}
```

#### 4.2.2 HttpRequestInfo - HTTP 请求信息（支持 API 重放）

```java
package com.houyu.common.log.model;

import lombok.Data;
import java.util.Map;

@Data
public class HttpRequestInfo {
    private String method;                     // HTTP 方法: GET/POST/PUT/DELETE 等
    private String uri;                        // 请求 URI
    private String url;                        // 完整 URL
    private String queryString;                // 查询参数
    private Map<String, String> headers;       // 请求头（脱敏后）
    private Map<String, String> cookies;       // Cookie（脱敏后）
    private String requestBody;                // 请求体（脱敏后）
    private Map<String, String[]> parameters;  // 请求参数
    private String contentType;                // Content-Type
    private String userAgent;                  // User-Agent
    private String referer;                    // Referer
}
```

#### 4.2.3 HttpResponseInfo - HTTP 响应信息（支持 API 重放）

```java
package com.houyu.common.log.model;

import lombok.Data;
import java.util.Map;

@Data
public class HttpResponseInfo {
    private Integer statusCode;                // HTTP 状态码
    private Map<String, String> headers;       // 响应头
    private String responseBody;               // 响应体（脱敏后）
    private String contentType;                // Content-Type
    private Long contentLength;                // 内容长度
}
```

### 4.3 链路追踪 ID 生成规则

#### 4.3.1 TraceId 生成策略

采用 **19位十进制数字** 作为全局唯一 TraceId 格式：

```
┌──────────────────┬──────────────────┬──────────────────┬──────────────┐
│   10位 时间戳     │    4位 机器码     │   4位 自增序列    │   1位 标志位  │
│  (yyMMddHHmm)    │  (Redis注册)      │  (进程内唯一)     │  (调用方传入) │
└──────────────────┴──────────────────┴──────────────────┴──────────────┘
```

**字段详细说明：**

| 字段 | 长度 | 格式/范围 | 说明 |
|------|------|-----------|------|
| **时间戳** | 10位 | `yyMMddHHmm` | 年月日时分格式，例如：2605131430 |
| **机器码** | 4位 | `0000~9999` | 整个集群唯一，服务启动时注册到 Redis |
| **自增序列** | 4位 | `0000~9999` | 进程内唯一，每分钟重置归零 |
| **标志位** | 1位 | `0~9` | 调用方传入（1~9），非法或不传默认为 0 |

**示例 TraceId：** `2605131430123456780`
- 时间戳: 2605131430 (2026年5月13日14:30)
- 机器码: 1234
- 自增序列: 5678
- 标志位: 0

#### 4.3.2 机器码注册机制（Redis）

```java
package com.houyu.common.log.trace;

import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import java.util.Collections;
import java.util.UUID;

@Component
public class MachineIdManager {
    
    private static final String MACHINE_ID_KEY = "log:trace:machine_id";
    private static final String MACHINE_LOCK_PREFIX = "log:trace:machine_lock:";
    private static final int MAX_MACHINE_ID = 9999;
    private static final long LOCK_EXPIRE_SECONDS = 60;
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    private String instanceId;
    private String machineId;
    
    @PostConstruct
    public void init() {
        instanceId = UUID.randomUUID().toString();
        machineId = registerMachineId();
    }
    
    /**
     * 注册机器码（使用 Lua 脚本保证原子性）
     */
    private String registerMachineId() {
        // Lua 脚本：查找第一个空闲的机器码并锁定
        String luaScript = 
            "for i = 0, 9999 do " +
            "  local lockKey = KEYS[1] .. tostring(i) " +
            "  if redis.call('setnx', lockKey, ARGV[1]) == 1 then " +
            "    redis.call('expire', lockKey, ARGV[2]) " +
            "    return tostring(i) " +
            "  end " +
            "end " +
            "return nil";
        
        DefaultRedisScript<String> script = new DefaultRedisScript<>();
        script.setScriptText(luaScript);
        script.setResultType(String.class);
        
        String result = redisTemplate.execute(script,
            Collections.singletonList(MACHINE_LOCK_PREFIX),
            instanceId,
            String.valueOf(LOCK_EXPIRE_SECONDS));
        
        if (result != null) {
            // 补零到4位
            return String.format("%04d", Integer.parseInt(result));
        }
        
        // 兜底：如果所有机器码都被占用，使用IP哈希取模
        return getFallbackMachineId();
    }
    
    /**
     * 兜底机器码生成策略
     */
    private String getFallbackMachineId() {
        try {
            String ip = InetAddress.getLocalHost().getHostAddress();
            int hash = Math.abs(ip.hashCode()) % 10000;
            return String.format("%04d", hash);
        } catch (Exception e) {
            // 终极兜底：随机数
            return String.format("%04d", (int)(Math.random() * 10000));
        }
    }
    
    /**
     * 定时续期锁（每分钟执行）
     */
    @Scheduled(fixedRate = 30000)
    public void renewLock() {
        if (machineId != null) {
            String lockKey = MACHINE_LOCK_PREFIX + machineId;
            redisTemplate.expire(lockKey, LOCK_EXPIRE_SECONDS, TimeUnit.SECONDS);
        }
    }
    
    @PreDestroy
    public void destroy() {
        // 服务停止时释放机器码
        if (machineId != null) {
            String lockKey = MACHINE_LOCK_PREFIX + machineId;
            String currentOwner = redisTemplate.opsForValue().get(lockKey);
            if (instanceId.equals(currentOwner)) {
                redisTemplate.delete(lockKey);
            }
        }
    }
    
    public String getMachineId() {
        return machineId;
    }
}
```

#### 4.3.3 自增序列设计（进程内唯一 + 兜底机制）

```java
package com.houyu.common.log.trace;

import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.ReentrantLock;

public class SequenceGenerator {
    
    private static final int MAX_SEQUENCE = 9999;
    
    // 主序列生成器
    private final AtomicInteger sequence = new AtomicInteger(0);
    
    // 兜底序列（当主序列溢出时使用）
    private final AtomicInteger fallbackSequence = new AtomicInteger(0);
    
    // 上一次重置的时间戳（yyMMddHHmm）
    private volatile String lastResetTimestamp = "";
    
    // 重置锁
    private final ReentrantLock resetLock = new ReentrantLock();
    
    /**
     * 获取下一个序列值
     * @param currentTimestamp 当前时间戳（yyMMddHHmm）
     */
    public int nextSequence(String currentTimestamp) {
        // 检查是否需要重置序列（新的一分钟）
        if (!currentTimestamp.equals(lastResetTimestamp)) {
            resetIfNeeded(currentTimestamp);
        }
        
        // 尝试使用主序列
        int current = sequence.getAndIncrement();
        if (current <= MAX_SEQUENCE) {
            return current;
        }
        
        // 主序列溢出，使用兜底序列（循环使用）
        int fallback = fallbackSequence.getAndIncrement() % (MAX_SEQUENCE + 1);
        return fallback;
    }
    
    private void resetIfNeeded(String currentTimestamp) {
        if (resetLock.tryLock()) {
            try {
                // 双重检查
                if (!currentTimestamp.equals(lastResetTimestamp)) {
                    sequence.set(0);
                    fallbackSequence.set(0);
                    lastResetTimestamp = currentTimestamp;
                }
            } finally {
                resetLock.unlock();
            }
        }
    }
}
```

#### 4.3.4 标志位处理逻辑

标志位来源：
1. 从 HTTP Header `X-Trace-Flag` 提取
2. 从 RPC 上下文 `traceFlag` 提取
3. 从线程上下文 `TraceContextHolder` 提取

处理规则：
- 传入值在 `1~9` 范围内：使用传入值
- 传入值不在范围内或未传入：默认使用 `0`

```java
public class FlagValidator {
    
    public static String normalizeFlag(String flag) {
        if (flag == null || flag.length() != 1) {
            return "0";
        }
        char c = flag.charAt(0);
        if (c >= '1' && c <= '9') {
            return flag;
        }
        return "0";
    }
}
```

#### 4.3.5 生成器完整实现

```java
package com.houyu.common.log.trace;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

@Component
public class DefaultTraceIdGenerator implements TraceIdGenerator {
    
    private static final DateTimeFormatter TIMESTAMP_FORMATTER = 
        DateTimeFormatter.ofPattern("yyMMddHHmm");
    
    @Autowired
    private MachineIdManager machineIdManager;
    
    private final SequenceGenerator sequenceGenerator = new SequenceGenerator();
    
    @Override
    public String generateTraceId() {
        return generateTraceId(null);
    }
    
    /**
     * 生成带标志位的 TraceId
     * @param flag 标志位（1~9）
     */
    public String generateTraceId(String flag) {
        // 1. 生成10位时间戳 (yyMMddHHmm)
        String timestamp = LocalDateTime.now().format(TIMESTAMP_FORMATTER);
        
        // 2. 获取4位机器码
        String machineId = machineIdManager.getMachineId();
        
        // 3. 获取4位自增序列
        int sequence = sequenceGenerator.nextSequence(timestamp);
        String sequenceStr = String.format("%04d", sequence);
        
        // 4. 处理1位标志位
        String flagStr = FlagValidator.normalizeFlag(flag);
        
        // 拼接19位 TraceId
        return timestamp + machineId + sequenceStr + flagStr;
    }
    
    @Override
    public String generateSpanId() {
        // SpanId 使用相同算法，但标志位固定为 9
        return generateTraceId("9");
    }
    
    @Override
    public String generateChildSpanId(String parentSpanId) {
        // 子 SpanId = 父 SpanId + 3位子序列
        int subSequence = sequenceGenerator.nextSequence(
            LocalDateTime.now().format(TIMESTAMP_FORMATTER)) % 1000;
        return parentSpanId + String.format("%03d", subSequence);
    }
}
```

#### 4.3.6 链路传递规则

1. **入站请求**：从 HTTP Header / RPC 上下文提取链路信息
   - `X-Trace-Id`: 链路追踪ID（19位）
   - `X-Span-Id`: 当前跨度ID
   - `X-Trace-Flag`: 标志位（1~9）
   - 存在 TraceId 则继续使用，否则生成新的 TraceId

2. **出站请求**：自动在 HTTP Header / RPC 上下文添加链路信息
   - `X-Trace-Id`: 当前链路 TraceId
   - `X-Span-Id`: 新生成的 SpanId
   - `X-Parent-Span-Id`: 当前 SpanId
   - `X-Trace-Flag`: 标志位（透传）

3. **MDC 集成**：自动将链路信息放入 MDC
   - `traceId`: 链路追踪ID
   - `spanId`: 当前跨度ID
   - `traceFlag`: 标志位

4. **线程上下文传递**：通过 `TraceContextHolder` 跨线程传递
   - 支持线程池场景（使用 `TraceableExecutorService` 包装）

### 4.4 日志格式规范

#### 4.4.1 JSON 格式（生产环境推荐）

```json
{
  "eventId": "event-xxx-yyy-zzz",
  "timestamp": "2026-05-13T10:30:00.000+08:00",
  "level": "INFO",
  "traceId": "1778654087010100421",
  "spanId": "1778654087010100439",
  "parentSpanId": null,
  "serviceName": "hy-user",
  "className": "com.houyu.user.service.UserService",
  "methodName": "getUserById",
  "message": "查询用户信息成功",
  "userId": "12345",
  "executionTime": 150,
  "isSuccess": true,
  "httpRequest": {
    "method": "GET",
    "uri": "/api/user/12345",
    "queryString": "?v=1",
    "headers": {
      "Content-Type": "application/json",
      "Authorization": "Bearer ***"
    }
  },
  "httpResponse": {
    "statusCode": 200,
    "contentType": "application/json"
  },
  "serverIp": "192.168.1.100",
  "clientIp": "10.0.0.50"
}
```

#### 4.4.2 文本格式（开发环境推荐）

```
[2026-05-13 10:30:00.000] [INFO] [traceId=1778654087010100421] [service=hy-user]
  com.houyu.user.service.UserService.getUserById(123) - 查询用户信息成功
  userId=12345, duration=150ms, clientIp=10.0.0.50
  GET /api/user/12345?v=1 → 200 OK
```

### 4.5 日志脱敏设计

#### 4.5.1 支持的脱敏类型

| 类型 | 示例 | 脱敏后 |
|------|------|--------|
| 手机号 | 13812345678 | 138****5678 |
| 邮箱 | test@example.com | te**@example.com |
| 身份证 | 110101199001011234 | 110***********1234 |
| 银行卡 | 6222021234567890123 | 6222***********0123 |
| 密码 | password123 | ****** |
| 手机号 | 13812345678 | 138****5678 |

#### 4.5.2 脱敏注解

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LogDesensitize {
    DesensitizeType type();
}
```

### 4.6 日志存储后置处理（DB 存储）

#### 4.6.1 数据库表设计

```sql
-- 日志主表
CREATE TABLE sys_log (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键ID',
    event_id VARCHAR(64) NOT NULL COMMENT '日志事件唯一ID',
    trace_id VARCHAR(64) NOT NULL COMMENT '链路追踪ID',
    span_id VARCHAR(64) COMMENT '当前跨度ID',
    parent_span_id VARCHAR(64) COMMENT '父跨度ID',
    
    service_name VARCHAR(64) COMMENT '服务名称',
    service_version VARCHAR(32) COMMENT '服务版本',
    environment VARCHAR(32) COMMENT '环境标识',
    
    log_level VARCHAR(16) NOT NULL COMMENT '日志级别',
    logger_name VARCHAR(512) COMMENT 'Logger名称',
    thread_name VARCHAR(128) COMMENT '线程名称',
    
    class_name VARCHAR(512) COMMENT '类名',
    method_name VARCHAR(128) COMMENT '方法名',
    file_name VARCHAR(256) COMMENT '文件名',
    line_number INT COMMENT '行号',
    
    message TEXT COMMENT '日志消息',
    formatted_message TEXT COMMENT '格式化后的消息',
    
    exception_class_name VARCHAR(512) COMMENT '异常类名',
    exception_message TEXT COMMENT '异常消息',
    stack_trace TEXT COMMENT '异常堆栈',
    
    user_id VARCHAR(64) COMMENT '用户ID',
    username VARCHAR(128) COMMENT '用户名',
    tenant_id VARCHAR(64) COMMENT '租户ID',
    
    server_ip VARCHAR(64) COMMENT '服务器IP',
    client_ip VARCHAR(64) COMMENT '客户端IP',
    
    execution_time BIGINT COMMENT '执行耗时(毫秒)',
    is_success TINYINT(1) COMMENT '是否成功',
    
    log_timestamp DATETIME NOT NULL COMMENT '日志时间戳',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    
    INDEX idx_trace_id (trace_id),
    INDEX idx_log_timestamp (log_timestamp),
    INDEX idx_user_id (user_id),
    INDEX idx_log_level (log_level),
    INDEX idx_service_name (service_name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='系统日志表';

-- HTTP 请求日志表（用于 API 重放）
CREATE TABLE sys_log_http_request (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键ID',
    log_id BIGINT NOT NULL COMMENT '关联日志ID',
    
    http_method VARCHAR(16) COMMENT 'HTTP方法',
    uri VARCHAR(1024) COMMENT '请求URI',
    url TEXT COMMENT '完整URL',
    query_string TEXT COMMENT '查询参数',
    
    headers TEXT COMMENT '请求头(JSON)',
    cookies TEXT COMMENT 'Cookie(JSON)',
    request_body LONGTEXT COMMENT '请求体',
    parameters TEXT COMMENT '请求参数(JSON)',
    
    content_type VARCHAR(256) COMMENT 'Content-Type',
    user_agent TEXT COMMENT 'User-Agent',
    referer TEXT COMMENT 'Referer',
    
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    INDEX idx_log_id (log_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='HTTP请求日志表';

-- HTTP 响应日志表（用于 API 重放）
CREATE TABLE sys_log_http_response (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键ID',
    log_id BIGINT NOT NULL COMMENT '关联日志ID',
    
    status_code INT COMMENT 'HTTP状态码',
    headers TEXT COMMENT '响应头(JSON)',
    response_body LONGTEXT COMMENT '响应体',
    content_type VARCHAR(256) COMMENT 'Content-Type',
    content_length BIGINT COMMENT '内容长度',
    
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    INDEX idx_log_id (log_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='HTTP响应日志表';
```

#### 4.6.2 DB 存储后置处理器

```java
package com.houyu.common.log.output;

public class DbLogOutput implements LogOutput {
    
    private final JdbcTemplate jdbcTemplate;
    private final AsyncDbLogWriter asyncWriter;
    
    @Override
    public void output(HyLogEvent logEvent) {
        // 异步写入数据库（避免影响性能）
        asyncWriter.write(logEvent);
    }
}
```

### 4.7 配置项设计

```yaml
hy:
  log:
    enabled: true                    # 是否启用日志模块
    format: json                     # 日志格式: json/text
    
    # Appender 配置
    appender:
      name: HY_COMMON_LOG
      buffer-size: 8192             # Disruptor 队列大小
    
    # 输出目标配置
    output:
      console:
        enabled: true                # 启用控制台输出
      file:
        enabled: true                # 启用文件输出
        path: /var/log/houyu         # 日志文件路径
        max-size: 100MB              # 单个文件最大大小
        max-history: 30              # 保留天数
      db:
        enabled: false               # 启用数据库存储（默认关闭）
        batch-size: 100              # 批量写入大小
        flush-interval: 5000         # 刷新间隔（毫秒）
        data-source: default         # 数据源名称
    
    # 脱敏配置
    desensitize:
      enabled: true                  # 是否启用脱敏
      patterns:                      # 自定义脱敏正则
        phone: "1[3-9]\\d{9}"
        email: "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"
    
    # 过滤配置
    filter:
      min-level: INFO               # 最低日志级别
      exclude-packages:             # 排除的包
        - org.springframework.*
        - com.alibaba.druid.*
    
    # 采样配置
    sampler:
      enabled: false                # 是否启用采样
      percentage: 10                # 采样百分比（0-100）
    
    # 链路追踪配置
    trace:
      enabled: true                  # 是否启用链路追踪
      generator: snowflake           # ID 生成器类型: snowflake/uuid
      data-center-id: 1              # 数据中心ID (0-31)
      worker-id: 1                   # 工作机器ID (0-31)
      header-names:                  # 链路头名称
        trace-id: X-Trace-Id
        span-id: X-Span-Id
        parent-span-id: X-Parent-Span-Id
    
    # API 重放配置
    replay:
      enabled: true                  # 是否启用 API 重放支持
      capture-request: true          # 是否捕获请求信息
      capture-response: true         # 是否捕获响应信息
      max-body-size: 1048576        # 最大捕获 Body 大小（字节）
      exclude-paths:                 # 不捕获的路径
        - /actuator/**
        - /health/**
```

## 5. 集成方式

### 5.1 依赖引入

```xml
<dependency>
    <groupId>com.houyu</groupId>
    <artifactId>hy-common-log</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

### 5.2 自动装配

通过 Spring Boot AutoConfiguration 自动装配，无需额外配置。

## 6. 扩展机制

### 6.1 自定义格式化器

```java
public interface LogFormatter {
    String format(HyLogEvent event);
}
```

### 6.2 自定义脱敏器

```java
public interface Desensitizer {
    String desensitize(String value);
    boolean support(String type);
}
```

## 7. 性能优化

- **异步日志处理**: 使用 LMAX Disruptor 高性能队列处理日志事件
- **批量写入**: 数据库存储采用批量写入机制
- **日志采样**: 高并发环境下可配置采样比例降低日志量
- **延迟格式化**: 避免不必要的字符串拼接
- **内存池**: 日志对象池化复用，减少 GC 压力

## 8. 版本历史

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.0.0 | 2026-05-13 | 修正 Logback 接管机制（使用 Appender），增加 LogEvent 模型、链路追踪、DB 存储、API 重放支持 |
