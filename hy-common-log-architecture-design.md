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
- 简化各业务模块的日志配置，无需修改 logback.xml
- 提供日志脱敏、过滤、链路追踪等增强功能
- 支持 API 重放所需的完整请求响应信息记录

## 2. 架构设计

### 2.1 整体架构图

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              业务应用模块                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │
│  │  hy-user │  │ hy-order │  │ hy-goods │  │  ...     │                  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘                  │
└───────┼─────────────┼─────────────┼─────────────┼──────────────────────────┘
        │             │             │             │
        │        SLF4J/Logback API                │
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
│  │                 Logback Appender 接管层                             │   │
│  │  ┌────────────────────────────────────────────────────────────┐   │   │
│  │  │  HyCommonLogAppender                                        │   │   │
│  │  │  - 实现 AppenderBase<ILoggingEvent>                         │   │   │
│  │  │  - prepareForDeferredProcessing() 后入 Disruptor 队列        │   │   │
│  │  │  - tryNext() 非阻塞投递，溢出丢弃并计 metrics               │   │   │
│  │  └──────────────────────┬─────────────────────────────────────┘   │   │
│  └─────────────────────────┼──────────────────────────────────────────┘   │
│                            │                                               │
│                            ▼                                               │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │                    LogEvent 转换层                                   │   │
│  │  LogEventConverter: ILoggingEvent → HyLogEvent                     │   │
│  │  （补充 MDC、traceId、请求信息等上下文）                            │   │
│  └─────────────────────────┬──────────────────────────────────────────┘   │
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
│              ┌─────────────┼──────────────┐                               │
│              ▼             ▼              ▼                                │
│  ┌─────────────────┐ ┌──────────┐ ┌───────────────────────────────────┐  │
│  │  Console（同步） │ │  File    │ │  DB（可选并行旁路，File写入后触发）│  │
│  │                 │ │  （异步） │ │  AsyncDbLogWriter（批量缓冲刷入）  │  │
│  └─────────────────┘ └──────────┘ └───────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件分层

| 层级 | 组件名称 | 职责说明 |
|------|---------|---------|
| 接管层 | HyCommonLogAppender | 实现 AppenderBase，调用 prepareForDeferredProcessing 后以 tryNext 投递 Disruptor |
| 转换层 | LogEventConverter | ILoggingEvent → HyLogEvent，补充上下文信息 |
| 增强层 | LogDesensitizer | 日志脱敏处理器 |
| 增强层 | LogFilter | 日志过滤器（按级别、关键词等） |
| 增强层 | LogSampler | 日志采样器，降低高并发下的日志量 |
| 格式化层 | LogFormatter | 日志格式化抽象接口 |
| 格式化层 | JsonLogFormatter | JSON 格式日志实现 |
| 格式化层 | TextLogFormatter | 文本格式日志实现 |
| 输出层 | ConsoleLogOutput | 控制台同步输出 |
| 输出层 | FileLogOutput | 文件异步输出 |
| 输出层 | DbLogOutput | DB 后置处理，委托 AsyncDbLogWriter 批量写入 |
| 输出层 | AsyncDbLogWriter | 批量缓冲 + 定时刷入，失败只计 metrics |
| 输出层 | LogTableRouter | 根据 traceId 前6位（yyMMdd）路由到对应日分表 |
| 链路层 | DefaultTraceIdGenerator | 生成19位 traceId（yyMMddHHmm+机器码+序列+标志） |
| 链路层 | MachineIdManager | 通过 JetCache + Redisson RLock 注册机器码到 Redis |
| 链路层 | SequenceGenerator | 进程内自增序列，每分钟重置 |
| 链路层 | TraceContextHolder | 管理 MDC 上下文，支持跨线程传递 |

## 3. 模块结构

### 3.1 目录结构

```
hy-common-log/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/houyu/common/log/
│   │   │       ├── config/
│   │   │       │   ├── LogAutoConfiguration.java
│   │   │       │   ├── RedissonConfig.java
│   │   │       │   └── LogProperties.java
│   │   │       ├── appender/
│   │   │       │   ├── HyCommonLogAppender.java
│   │   │       │   ├── LogEventHolder.java
│   │   │       │   └── LogEventHandler.java
│   │   │       ├── converter/
│   │   │       │   └── LogEventConverter.java
│   │   │       ├── model/
│   │   │       │   ├── HyLogEvent.java
│   │   │       │   ├── HttpRequestInfo.java
│   │   │       │   ├── HttpResponseInfo.java
│   │   │       │   └── LogLevel.java
│   │   │       ├── formatter/
│   │   │       │   ├── LogFormatter.java
│   │   │       │   ├── JsonLogFormatter.java
│   │   │       │   └── TextLogFormatter.java
│   │   │       ├── desensitizer/
│   │   │       │   ├── Desensitizer.java
│   │   │       │   ├── PhoneDesensitizer.java
│   │   │       │   ├── EmailDesensitizer.java
│   │   │       │   ├── IdCardDesensitizer.java
│   │   │       │   ├── BankCardDesensitizer.java
│   │   │       │   ├── PasswordDesensitizer.java
│   │   │       │   └── DesensitizerManager.java
│   │   │       ├── filter/
│   │   │       │   ├── LogFilter.java
│   │   │       │   ├── LevelLogFilter.java
│   │   │       │   └── KeywordLogFilter.java
│   │   │       ├── sampler/
│   │   │       │   ├── LogSampler.java
│   │   │       │   └── PercentageLogSampler.java
│   │   │       ├── output/
│   │   │       │   ├── LogOutput.java
│   │   │       │   ├── LogOutputManager.java
│   │   │       │   ├── ConsoleLogOutput.java
│   │   │       │   ├── FileLogOutput.java
│   │   │       │   ├── DbLogOutput.java
│   │   │       │   ├── AsyncDbLogWriter.java
│   │   │       │   └── LogTableRouter.java
│   │   │       ├── trace/
│   │   │       │   ├── TraceIdGenerator.java
│   │   │       │   ├── DefaultTraceIdGenerator.java
│   │   │       │   ├── MachineIdManager.java
│   │   │       │   ├── SequenceGenerator.java
│   │   │       │   ├── FlagValidator.java
│   │   │       │   └── TraceContextHolder.java
│   │   │       ├── annotation/
│   │   │       │   └── LogDesensitize.java
│   │   │       ├── constant/
│   │   │       │   └── LogConstants.java
│   │   │       └── util/
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
│               ├── trace/
│               │   ├── DefaultTraceIdGeneratorTest.java
│               │   └── SequenceGeneratorTest.java
│               ├── output/
│               │   └── LogTableRouterTest.java
│               └── appender/
│                   └── HyCommonLogAppenderTest.java
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

        <!-- Spring Boot JDBC（DB 存储，可选） -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Logback（Spring Boot 默认包含） -->
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
        </dependency>

        <!-- Disruptor 高性能队列（异步日志处理） -->
        <dependency>
            <groupId>com.lmax</groupId>
            <artifactId>disruptor</artifactId>
            <version>3.4.4</version>
        </dependency>

        <!-- Jackson JSON 格式化 -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>

        <!-- JetCache（本地+远程缓存，机器码注册，可选） -->
        <dependency>
            <groupId>com.alicp.jetcache</groupId>
            <artifactId>jetcache-starter-redis-lettuce</artifactId>
            <version>2.7.3</version>
            <optional>true</optional>
        </dependency>

        <!-- Redisson（分布式锁，机器码注册，可选） -->
        <dependency>
            <groupId>org.redisson</groupId>
            <artifactId>redisson-spring-boot-starter</artifactId>
            <version>3.23.5</version>
            <optional>true</optional>
        </dependency>

        <!-- Micrometer（metrics 指标暴露） -->
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-core</artifactId>
            <optional>true</optional>
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

通过 `LogAutoConfiguration` 在 Spring 启动时以编程方式将 `HyCommonLogAppender` 注册到 Logback 的 root logger，业务模块无需修改 `logback-spring.xml`。

#### 4.1.2 HyCommonLogAppender 实现

```java
package com.houyu.common.log.appender;

import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.core.AppenderBase;
import com.lmax.disruptor.InsufficientCapacityException;
import com.lmax.disruptor.RingBuffer;
import com.lmax.disruptor.dsl.Disruptor;
import com.lmax.disruptor.dsl.ProducerType;
import com.lmax.disruptor.BlockingWaitStrategy;
import io.micrometer.core.instrument.Metrics;

import java.util.concurrent.ThreadFactory;
import java.util.concurrent.atomic.AtomicLong;

public class HyCommonLogAppender extends AppenderBase<ILoggingEvent> {

    private Disruptor<LogEventHolder> disruptor;
    private RingBuffer<LogEventHolder> ringBuffer;
    private final int bufferSize;
    private final AtomicLong droppedCount = new AtomicLong(0);

    public HyCommonLogAppender(int bufferSize) {
        this.bufferSize = bufferSize;
    }

    @Override
    public void start() {
        ThreadFactory threadFactory = r -> {
            Thread t = new Thread(r, "log-disruptor");
            t.setDaemon(true);
            return t;
        };
        disruptor = new Disruptor<>(
                LogEventHolder::new,
                bufferSize,
                threadFactory,
                ProducerType.MULTI,
                new BlockingWaitStrategy()
        );
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
        // 必须在发布前调用，防止异步处理时 logback 复用或清空 event 对象
        eventObject.prepareForDeferredProcessing();

        try {
            long sequence = ringBuffer.tryNext();
            try {
                LogEventHolder holder = ringBuffer.get(sequence);
                holder.setLoggingEvent(eventObject);
            } finally {
                ringBuffer.publish(sequence);
            }
        } catch (InsufficientCapacityException e) {
            // 队列满时丢弃，记录指标，不阻塞业务线程
            long dropped = droppedCount.incrementAndGet();
            Metrics.counter("hy.log.dropped").increment();
            if (dropped % 1000 == 0) {
                addWarn("Log queue full, total dropped: " + dropped);
            }
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

#### 4.1.3 LogEventHolder 事件封装

```java
package com.houyu.common.log.appender;

import ch.qos.logback.classic.spi.ILoggingEvent;

public class LogEventHolder {
    private ILoggingEvent loggingEvent;

    public ILoggingEvent getLoggingEvent() { return loggingEvent; }
    public void setLoggingEvent(ILoggingEvent e) { this.loggingEvent = e; }
}
```

#### 4.1.4 LogEventHandler 事件处理器

```java
package com.houyu.common.log.appender;

import com.houyu.common.log.converter.LogEventConverter;
import com.houyu.common.log.model.HyLogEvent;
import com.houyu.common.log.output.LogOutputManager;
import com.lmax.disruptor.EventHandler;

public class LogEventHandler implements EventHandler<LogEventHolder> {

    private final LogEventConverter converter;
    private final LogOutputManager outputManager;

    public LogEventHandler(LogEventConverter converter, LogOutputManager outputManager) {
        this.converter = converter;
        this.outputManager = outputManager;
    }

    @Override
    public void onEvent(LogEventHolder holder, long sequence, boolean endOfBatch) {
        HyLogEvent hyLogEvent = converter.convert(holder.getLoggingEvent());
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
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Configuration;

import jakarta.annotation.PostConstruct;

@Configuration
@EnableConfigurationProperties(LogProperties.class)
@ConditionalOnProperty(prefix = "hy.log", name = "enabled", havingValue = "true", matchIfMissing = true)
public class LogAutoConfiguration {

    private final LogProperties logProperties;

    public LogAutoConfiguration(LogProperties logProperties) {
        this.logProperties = logProperties;
    }

    @PostConstruct
    public void init() {
        LoggerContext loggerContext = (LoggerContext) LoggerFactory.getILoggerFactory();

        HyCommonLogAppender appender = new HyCommonLogAppender(
                logProperties.getAppender().getBufferSize()
        );
        appender.setContext(loggerContext);
        appender.setName("HY_COMMON_LOG");
        appender.start();

        Logger rootLogger = loggerContext.getLogger(Logger.ROOT_LOGGER_NAME);
        // 避免重复注册（热重启场景）
        if (rootLogger.getAppender("HY_COMMON_LOG") == null) {
            rootLogger.addAppender(appender);
        }
    }
}
```

#### 4.1.6 RedissonConfig（条件注入，避免与业务模块冲突）

```java
package com.houyu.common.log.config;

import org.redisson.Redisson;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@ConditionalOnProperty(prefix = "hy.log.trace", name = "enabled", havingValue = "true", matchIfMissing = true)
public class RedissonConfig {

    // 仅在业务模块未提供 RedissonClient Bean 时才创建
    @Bean
    @ConditionalOnMissingBean(RedissonClient.class)
    public RedissonClient redissonClient(LogProperties logProperties) {
        Config config = new Config();
        config.useSingleServer()
              .setAddress(logProperties.getTrace().getRedisAddress())
              .setDatabase(logProperties.getTrace().getRedisDatabase());
        return Redisson.create(config);
    }
}
```

### 4.2 统一 LogEvent 模型

#### 4.2.1 HyLogEvent

```java
package com.houyu.common.log.model;

import lombok.Data;
import java.time.LocalDateTime;
import java.util.Map;

@Data
public class HyLogEvent {

    // ========== 基础日志字段 ==========
    private String eventId;                 // 日志事件唯一ID（19位，同 traceId 格式）
    private LocalDateTime timestamp;        // 日志时间戳
    private LogLevel level;                 // 日志级别
    private String loggerName;              // Logger 名称
    private String threadName;              // 线程名称
    private String message;                 // 日志消息
    private String formattedMessage;        // 格式化后的消息

    // ========== 异常信息 ==========
    private String exceptionClassName;      // 异常类名
    private String exceptionMessage;        // 异常消息
    private String stackTrace;              // 异常堆栈

    // ========== 调用位置信息 ==========
    private String className;               // 类名
    private String methodName;              // 方法名
    private String fileName;                // 文件名
    private Integer lineNumber;             // 行号

    // ========== 链路追踪字段 ==========
    private String traceId;                 // 链路追踪ID（19位）
    private String spanId;                  // 当前跨度ID（19位）
    private String parentSpanId;            // 父跨度ID（19位）
    private String serviceName;             // 服务名称
    private String serviceVersion;          // 服务版本
    private String environment;             // 环境标识（dev/test/prod）

    // ========== API 重放支持字段 ==========
    private HttpRequestInfo httpRequest;    // HTTP 请求信息
    private HttpResponseInfo httpResponse;  // HTTP 响应信息
    private Long executionTime;             // 执行耗时（毫秒）
    private Boolean success;                // 是否成功
    private Boolean replayable;             // 是否可重放
    private String requestHash;             // 请求签名（幂等校验）
    private Integer replayCount;            // 已重放次数

    // ========== 用户/租户信息 ==========
    private String userId;                  // 用户ID
    private String username;                // 用户名
    private String tenantId;                // 租户ID

    // ========== 系统信息 ==========
    private String serverIp;                // 服务器IP
    private String clientIp;               // 客户端IP
    private Map<String, String> mdcContext; // MDC 上下文

    // ========== 扩展字段 ==========
    private Map<String, Object> extensions; // 自定义扩展字段
}
```

#### 4.2.2 HttpRequestInfo

```java
package com.houyu.common.log.model;

import lombok.Data;
import java.util.Map;

@Data
public class HttpRequestInfo {
    private String method;                      // HTTP 方法
    private String uri;                         // 请求 URI
    private String url;                         // 完整 URL
    private String queryString;                 // 查询参数
    private Map<String, String> headers;        // 请求头（脱敏后）
    private Map<String, String> cookies;        // Cookie（脱敏后）
    private String requestBody;                 // 请求体（脱敏后）
    private Map<String, String[]> parameters;   // 请求参数
    private String contentType;                 // Content-Type
    private String userAgent;                   // User-Agent
    private String referer;                     // Referer
    private String protocol;                    // HTTP/1.1、HTTP/2（重放时还原协议）
}
```

#### 4.2.3 HttpResponseInfo

```java
package com.houyu.common.log.model;

import lombok.Data;
import java.util.Map;

@Data
public class HttpResponseInfo {
    private Integer statusCode;             // HTTP 状态码
    private Map<String, String> headers;    // 响应头
    private String responseBody;            // 响应体（脱敏后）
    private String contentType;             // Content-Type
    private Long contentLength;             // 内容长度
    private String errorCode;              // 业务错误码
    private String errorMessage;           // 业务错误消息
}
```

### 4.3 链路追踪 ID 生成规则

#### 4.3.1 TraceId 格式（19位十进制字符串）

```
┌──────────────────┬──────────────┬──────────────┬──────┐
│   10位 时间戳     │  4位 机器码  │  4位 自增序列 │ 1位  │
│  (yyMMddHHmm)    │  (0000~9999) │  (0000~9999) │ 标志 │
└──────────────────┴──────────────┴──────────────┴──────┘
```

| 字段 | 长度 | 格式/范围 | 说明 |
|------|------|-----------|------|
| 时间戳 | 10位 | `yyMMddHHmm` | 年月日时分，例：2605131430 |
| 机器码 | 4位 | `0000~9999` | 集群唯一，启动时通过 Redis 注册 |
| 自增序列 | 4位 | `0000~9999` | 进程内唯一，每分钟重置归零 |
| 标志位 | 1位 | `0~9` | 调用方传入 1~9，非法或不传默认为 0 |

**示例：** `2605131430001200010`
- 时间戳：`2605131430`（2026-05-13 14:30）
- 机器码：`0012`
- 自增序列：`0001`
- 标志位：`0`

**吞吐上限：** 单机每分钟最多生成 9999 条唯一 traceId。超出则通过兜底机制循环复用，配合时间戳仍可区分。

#### 4.3.2 机器码注册机制（JetCache + Redisson RLock）

**设计要点：**
- 使用 Redisson `RLock`（不指定 leaseTime）启用看门狗，进程存活期间锁自动续期
- 使用 JetCache `CacheType.REMOTE`（Redis）存储已分配的机器码映射，跨实例共享
- `@PreDestroy` 优雅停机时释放锁和缓存，机器码立即可被复用

```java
package com.houyu.common.log.trace;

import com.alicp.jetcache.Cache;
import com.alicp.jetcache.anno.CacheType;
import com.alicp.jetcache.anno.CreateCache;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.boot.autoconfigure.condition.ConditionalOnBean;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.stereotype.Component;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import java.net.InetAddress;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

@Component
@EnableScheduling
@ConditionalOnBean(RedissonClient.class)
public class MachineIdManager {

    private static final String MACHINE_LOCK_PREFIX = "log:trace:{machine_lock}:";
    private static final int MAX_MACHINE_ID = 9999;
    private static final long LOCK_WAIT_SECONDS = 0; // 非阻塞，快速遍历
    // 不指定 leaseTime，启用 Redisson 看门狗自动续期

    // CacheType.REMOTE：存入 Redis，所有实例共享已分配的机器码视图
    @CreateCache(name = "log:trace:allocated_machines",
                 cacheType = CacheType.REMOTE,
                 expire = 300,
                 timeUnit = TimeUnit.SECONDS)
    private Cache<Integer, String> allocatedMachines;

    private final RedissonClient redissonClient;
    private String instanceId;
    private volatile String machineId;
    private volatile RLock heldLock; // 当前实例持有的锁

    public MachineIdManager(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }

    @PostConstruct
    public void init() {
        instanceId = UUID.randomUUID().toString();
        machineId = registerMachineId();
    }

    private String registerMachineId() {
        for (int i = 0; i <= MAX_MACHINE_ID; i++) {
            String lockKey = MACHINE_LOCK_PREFIX + i;
            RLock lock = redissonClient.getLock(lockKey);
            boolean acquired = false;
            try {
                // tryLock 不指定 leaseTime → 看门狗自动续期
                acquired = lock.tryLock(LOCK_WAIT_SECONDS, TimeUnit.SECONDS);
                if (acquired) {
                    String owner = allocatedMachines.get(i);
                    if (owner == null) {
                        // 占用该机器码：写入 Redis，保存锁引用
                        allocatedMachines.put(i, instanceId);
                        heldLock = lock;
                        return String.format("%04d", i);
                    }
                    // 已被占用：释放锁，尝试下一个
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            } finally {
                // 未认领该槽位时必须释放锁
                if (acquired && heldLock != lock) {
                    try { lock.unlock(); } catch (Exception ignored) {}
                }
            }
        }
        return getFallbackMachineId();
    }

    private String getFallbackMachineId() {
        try {
            String ip = InetAddress.getLocalHost().getHostAddress();
            int hash = Math.abs(ip.hashCode()) % 10000;
            return String.format("%04d", hash);
        } catch (Exception e) {
            return String.format("%04d", (int) (Math.random() * 10000));
        }
    }

    @PreDestroy
    public void destroy() {
        if (heldLock != null) {
            try {
                if (heldLock.isHeldByCurrentThread()) {
                    heldLock.unlock();
                }
            } catch (Exception ignored) {}
        }
        if (machineId != null) {
            try {
                allocatedMachines.remove(Integer.parseInt(machineId));
            } catch (Exception ignored) {}
        }
    }

    public String getMachineId() {
        return machineId != null ? machineId : "0000";
    }
}
```

**JetCache + Redisson 配置参考：**

```yaml
jetcache:
  local:
    default:
      type: caffeine
      limit: 10000
  remote:
    default:
      type: redis.lettuce       # JetCache 远程存储用 Lettuce
      keyConvertor: fastjson2
      valueEncoder: java
      valueDecoder: java
      uri: redis://localhost:6379

# Redisson（分布式锁）：使用项目已有配置，无需重复配置
# 若项目未配置 Redisson，模块将创建默认 RedissonClient（见 RedissonConfig）
```

#### 4.3.3 自增序列设计（进程内唯一 + 兜底）

```java
package com.houyu.common.log.trace;

import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.ReentrantLock;

public class SequenceGenerator {

    private static final int MAX_SEQUENCE = 9999;
    private final AtomicInteger sequence = new AtomicInteger(0);
    private volatile String lastResetTimestamp = "";
    private final ReentrantLock resetLock = new ReentrantLock();

    public int nextSequence(String currentTimestamp) {
        if (!currentTimestamp.equals(lastResetTimestamp)) {
            resetIfNeeded(currentTimestamp);
        }
        // Math.abs 防止极小概率 Integer 溢出变负数
        return Math.abs(sequence.getAndIncrement()) % (MAX_SEQUENCE + 1);
    }

    private void resetIfNeeded(String currentTimestamp) {
        if (resetLock.tryLock()) {
            try {
                if (!currentTimestamp.equals(lastResetTimestamp)) {
                    sequence.set(0);
                    lastResetTimestamp = currentTimestamp;
                }
            } finally {
                resetLock.unlock();
            }
        }
        // 未获取锁的线程继续使用当前计数器，同一分钟内序列不重置无影响
    }
}
```

#### 4.3.4 标志位处理

| 来源 | 优先级 |
|------|--------|
| HTTP Header `X-Trace-Flag` | 1（最高） |
| RPC 上下文 `traceFlag` | 2 |
| `TraceContextHolder` 线程上下文 | 3 |
| 默认值 | `0` |

```java
public class FlagValidator {
    public static String normalizeFlag(String flag) {
        if (flag != null && flag.length() == 1) {
            char c = flag.charAt(0);
            if (c >= '1' && c <= '9') {
                return flag;
            }
        }
        return "0";
    }
}
```

#### 4.3.5 TraceId 生成器完整实现

```java
package com.houyu.common.log.trace;

import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

@Component
public class DefaultTraceIdGenerator implements TraceIdGenerator {

    private static final DateTimeFormatter FORMATTER =
            DateTimeFormatter.ofPattern("yyMMddHHmm");

    private final MachineIdManager machineIdManager;
    private final SequenceGenerator sequenceGenerator = new SequenceGenerator();

    public DefaultTraceIdGenerator(MachineIdManager machineIdManager) {
        this.machineIdManager = machineIdManager;
    }

    @Override
    public String generateTraceId() {
        return generate(null);
    }

    @Override
    public String generateTraceId(String flag) {
        return generate(flag);
    }

    @Override
    public String generateSpanId() {
        return generate(null);
    }

    @Override
    public String generateChildSpanId(String parentSpanId) {
        // 子 SpanId 是独立的19位 ID，父子关系通过 HyLogEvent.parentSpanId 字段维护
        return generate(null);
    }

    private String generate(String flag) {
        String timestamp = LocalDateTime.now().format(FORMATTER);
        String machineId = machineIdManager.getMachineId();
        String seq = String.format("%04d", sequenceGenerator.nextSequence(timestamp));
        String flagStr = FlagValidator.normalizeFlag(flag);
        return timestamp + machineId + seq + flagStr;
    }
}
```

#### 4.3.6 链路传递规则

1. **入站请求**（优先级：MDC > HTTP Header > 本地生成）
   - 从 `X-Trace-Id` / `X-Span-Id` / `X-Trace-Flag` 提取，放入 MDC
   - 不存在则调用 `generateTraceId(flag)` 生成新 ID

2. **出站请求**
   - 透传 `X-Trace-Id`，生成新 `X-Span-Id`，将当前 SpanId 设为 `X-Parent-Span-Id`
   - 透传 `X-Trace-Flag`

3. **异步线程**
   - 使用 `TransmittableThreadLocal`（TTL）替代普通 `ThreadLocal`，防止线程池场景下 MDC 丢失

### 4.4 日志格式规范

#### 4.4.1 JSON 格式（生产环境推荐）

```json
{
  "eventId": "2605131430001200010",
  "timestamp": "2026-05-13T14:30:00.000+08:00",
  "level": "INFO",
  "traceId": "2605131430001200010",
  "spanId": "2605131430001200020",
  "parentSpanId": null,
  "serviceName": "hy-user",
  "className": "com.houyu.user.service.UserService",
  "methodName": "getUserById",
  "message": "查询用户信息成功",
  "userId": "12345",
  "executionTime": 150,
  "success": true,
  "replayable": true,
  "requestHash": "a1b2c3d4e5f6",
  "httpRequest": {
    "method": "GET",
    "uri": "/api/user/12345",
    "queryString": "?v=1",
    "protocol": "HTTP/1.1",
    "headers": {
      "Content-Type": "application/json",
      "Authorization": "Bearer ***",
      "X-Trace-Flag": "5"
    }
  },
  "httpResponse": {
    "statusCode": 200,
    "contentType": "application/json",
    "errorCode": null,
    "errorMessage": null
  },
  "serverIp": "192.168.1.100",
  "clientIp": "10.0.0.50"
}
```

#### 4.4.2 文本格式（开发环境推荐）

```
[2026-05-13 14:30:00.000] [INFO] [traceId=2605131430001200010] [service=hy-user]
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

#### 4.5.2 脱敏注解

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LogDesensitize {
    DesensitizeType type();
}
```

**运行时触发机制：**
- **注解脱敏**：自定义 Jackson `JsonSerializer`，在对象序列化时扫描 `@LogDesensitize` 字段，仅在日志输出阶段生效，不影响业务对象本身
- **正则兜底**：`DesensitizerManager` 对最终 message 字符串执行正则替换，作为字符串日志的兜底

### 4.6 日志存储后置处理（DB 存储）

#### 4.6.1 后置处理架构

DB 存储是 **File 写入后的可选并行旁路**：
- 主流程：`Disruptor → 格式化 → File 输出`（不受 DB 影响）
- 旁路：File 写入完成后，`DbLogOutput` 将事件投递至 `AsyncDbLogWriter` 内存队列
- DB 写入失败时：仅递增 `hy.log.db.failed` 指标，不抛异常，不影响主流程
- 内存队列满时：丢弃并递增 `hy.log.db.dropped` 指标

#### 4.6.2 AsyncDbLogWriter（批量缓冲 + 定时刷入）

```java
package com.houyu.common.log.output;

import com.houyu.common.log.model.HyLogEvent;
import io.micrometer.core.instrument.Metrics;
import org.springframework.jdbc.core.JdbcTemplate;

import jakarta.annotation.PreDestroy;
import java.util.*;
import java.util.concurrent.*;
import java.util.stream.Collectors;

public class AsyncDbLogWriter {

    private static final int QUEUE_CAPACITY = 65536;

    private final BlockingQueue<HyLogEvent> queue = new LinkedBlockingQueue<>(QUEUE_CAPACITY);
    private final JdbcTemplate jdbcTemplate;
    private final LogTableRouter tableRouter;
    private final int batchSize;
    private final ScheduledExecutorService scheduler;

    public AsyncDbLogWriter(JdbcTemplate jdbcTemplate, LogTableRouter tableRouter,
                            int batchSize, long flushIntervalMs) {
        this.jdbcTemplate = jdbcTemplate;
        this.tableRouter = tableRouter;
        this.batchSize = batchSize;
        this.scheduler = Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "log-db-writer");
            t.setDaemon(true);
            return t;
        });
        scheduler.scheduleAtFixedRate(this::flush, flushIntervalMs, flushIntervalMs,
                TimeUnit.MILLISECONDS);
    }

    public void write(HyLogEvent event) {
        if (!queue.offer(event)) {
            Metrics.counter("hy.log.db.dropped").increment();
        }
    }

    private void flush() {
        List<HyLogEvent> batch = new ArrayList<>(batchSize);
        queue.drainTo(batch, batchSize);
        if (batch.isEmpty()) return;

        // 按目标表分组（同一天的日志写入同一张分表）
        Map<String, List<HyLogEvent>> byTable = batch.stream()
                .collect(Collectors.groupingBy(
                        e -> tableRouter.resolveTableName("sys_log", e.getTraceId())));

        byTable.forEach((tableName, events) -> {
            try {
                tableRouter.ensureTableExists(tableName);
                batchInsert(tableName, events);
            } catch (Exception ex) {
                Metrics.counter("hy.log.db.failed").increment(events.size());
                // 非致命，不向上抛出
            }
        });
    }

    private void batchInsert(String tableName, List<HyLogEvent> events) {
        String sql = "INSERT INTO " + tableName +
                " (event_id, trace_id, span_id, parent_span_id, service_name, log_level," +
                "  message, user_id, client_ip, server_ip, execution_time, is_success," +
                "  log_timestamp, created_at)" +
                " VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,CURRENT_TIMESTAMP)";
        jdbcTemplate.batchUpdate(sql, events, events.size(), (ps, e) -> {
            ps.setString(1, e.getEventId());
            ps.setString(2, e.getTraceId());
            ps.setString(3, e.getSpanId());
            ps.setString(4, e.getParentSpanId());
            ps.setString(5, e.getServiceName());
            ps.setString(6, e.getLevel() != null ? e.getLevel().name() : null);
            ps.setString(7, e.getMessage());
            ps.setString(8, e.getUserId());
            ps.setString(9, e.getClientIp());
            ps.setString(10, e.getServerIp());
            ps.setObject(11, e.getExecutionTime());
            ps.setObject(12, e.getSuccess());
            ps.setObject(13, e.getTimestamp());
        });
    }

    @PreDestroy
    public void shutdown() {
        scheduler.shutdown();
        // 优雅停机时刷出剩余数据
        flush();
    }
}
```

#### 4.6.3 LogTableRouter（按 traceId 前6位路由到日分表）

```java
package com.houyu.common.log.output;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.util.Collections;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class LogTableRouter {

    private static final DateTimeFormatter DATE_FMT = DateTimeFormatter.ofPattern("yyyyMMdd");
    // 已确认存在的表名缓存，避免重复 DDL 检查
    private final Set<String> existingTables = Collections.newSetFromMap(new ConcurrentHashMap<>());
    private final JdbcTemplate jdbcTemplate;

    public LogTableRouter(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    /**
     * 根据 traceId 解析目标日分表名。
     * traceId 格式：yyMMddHHmm（前10位），其中前6位 yyMMdd 即日期。
     */
    public String resolveTableName(String baseTable, String traceId) {
        if (traceId != null && traceId.length() >= 6) {
            String yyMMdd = traceId.substring(0, 6);
            return baseTable + "_20" + yyMMdd; // 如 sys_log_20260513
        }
        return baseTable + "_" + LocalDate.now().format(DATE_FMT);
    }

    /**
     * 确保目标表存在（LIKE 模板表复制结构）。
     * PostgreSQL：CREATE TABLE IF NOT EXISTS ... (LIKE base_table INCLUDING ALL)
     */
    public void ensureTableExists(String tableName) {
        if (existingTables.contains(tableName)) return;
        String baseTable = tableName.replaceAll("_\\d{8}$", "");
        jdbcTemplate.execute(
                "CREATE TABLE IF NOT EXISTS " + tableName +
                " (LIKE " + baseTable + " INCLUDING ALL)"
        );
        existingTables.add(tableName);
    }

    /** 每天 23:55 预建次日所有分表 */
    @Scheduled(cron = "0 55 23 * * *")
    public void preCreateNextDayTables() {
        String nextDay = LocalDate.now().plusDays(1).format(DATE_FMT);
        ensureTableExists("sys_log_" + nextDay);
        ensureTableExists("sys_log_http_request_" + nextDay);
        ensureTableExists("sys_log_http_response_" + nextDay);
    }

    /** 每天 02:00 清理超过保留期的历史分表（默认保留30天） */
    @Scheduled(cron = "0 0 2 * * *")
    public void cleanOldTables() {
        LocalDate cutoff = LocalDate.now().minusDays(30);
        String cutoffStr = "20" + cutoff.format(DateTimeFormatter.ofPattern("yyMMdd"));
        // 查询 information_schema 中匹配模式的分表并 DROP
        jdbcTemplate.queryForList(
                "SELECT table_name FROM information_schema.tables " +
                "WHERE table_schema = current_schema() AND table_name ~ '^sys_log_\\d{8}$'",
                String.class
        ).stream()
         .filter(name -> {
             String datePart = name.replaceAll("^sys_log_", "");
             return datePart.compareTo(cutoffStr) < 0;
         })
         .forEach(name -> jdbcTemplate.execute("DROP TABLE IF EXISTS " + name));
    }
}
```

#### 4.6.4 数据库表设计（PostgreSQL）

```sql
-- ============================================================
-- 日志主表模板（实际数据存入日分表，如 sys_log_20260513）
-- ============================================================
CREATE TABLE sys_log (
    id                  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    event_id            VARCHAR(19)     NOT NULL,
    trace_id            VARCHAR(19)     NOT NULL,
    span_id             VARCHAR(19),
    parent_span_id      VARCHAR(19),
    service_name        VARCHAR(64),
    service_version     VARCHAR(32),
    environment         VARCHAR(32),
    log_level           VARCHAR(16)     NOT NULL,
    logger_name         VARCHAR(512),
    thread_name         VARCHAR(128),
    class_name          VARCHAR(512),
    method_name         VARCHAR(128),
    file_name           VARCHAR(256),
    line_number         INT,
    message             TEXT,
    formatted_message   TEXT,
    exception_class_name VARCHAR(512),
    exception_message   TEXT,
    stack_trace         TEXT,
    user_id             VARCHAR(64),
    username            VARCHAR(128),
    tenant_id           VARCHAR(64),
    server_ip           VARCHAR(64),
    client_ip           VARCHAR(64),
    execution_time      BIGINT,
    is_success          BOOLEAN,
    log_timestamp       TIMESTAMPTZ     NOT NULL,
    created_at          TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP
);

COMMENT ON TABLE  sys_log                       IS '系统日志表模板（按天分表：sys_log_yyyyMMdd）';
COMMENT ON COLUMN sys_log.id                    IS '主键ID';
COMMENT ON COLUMN sys_log.event_id              IS '日志事件唯一ID（19位）';
COMMENT ON COLUMN sys_log.trace_id              IS '链路追踪ID（19位，yyMMddHHmm+机器码+序列+标志）';
COMMENT ON COLUMN sys_log.span_id               IS '当前跨度ID（19位）';
COMMENT ON COLUMN sys_log.parent_span_id        IS '父跨度ID（19位）';
COMMENT ON COLUMN sys_log.service_name          IS '服务名称';
COMMENT ON COLUMN sys_log.service_version       IS '服务版本';
COMMENT ON COLUMN sys_log.environment           IS '环境标识（dev/test/prod）';
COMMENT ON COLUMN sys_log.log_level             IS '日志级别';
COMMENT ON COLUMN sys_log.logger_name           IS 'Logger名称';
COMMENT ON COLUMN sys_log.thread_name           IS '线程名称';
COMMENT ON COLUMN sys_log.class_name            IS '类名';
COMMENT ON COLUMN sys_log.method_name           IS '方法名';
COMMENT ON COLUMN sys_log.file_name             IS '文件名';
COMMENT ON COLUMN sys_log.line_number           IS '行号';
COMMENT ON COLUMN sys_log.message               IS '日志消息';
COMMENT ON COLUMN sys_log.formatted_message     IS '格式化后的消息';
COMMENT ON COLUMN sys_log.exception_class_name  IS '异常类名';
COMMENT ON COLUMN sys_log.exception_message     IS '异常消息';
COMMENT ON COLUMN sys_log.stack_trace           IS '异常堆栈';
COMMENT ON COLUMN sys_log.user_id               IS '用户ID';
COMMENT ON COLUMN sys_log.username              IS '用户名';
COMMENT ON COLUMN sys_log.tenant_id             IS '租户ID';
COMMENT ON COLUMN sys_log.server_ip             IS '服务器IP';
COMMENT ON COLUMN sys_log.client_ip             IS '客户端IP';
COMMENT ON COLUMN sys_log.execution_time        IS '执行耗时（毫秒）';
COMMENT ON COLUMN sys_log.is_success            IS '是否成功';
COMMENT ON COLUMN sys_log.log_timestamp         IS '日志时间戳';
COMMENT ON COLUMN sys_log.created_at            IS '创建时间';

CREATE INDEX idx_sys_log_trace_id       ON sys_log (trace_id);
CREATE INDEX idx_sys_log_log_timestamp  ON sys_log (log_timestamp);
CREATE INDEX idx_sys_log_user_id        ON sys_log (user_id);
CREATE INDEX idx_sys_log_log_level      ON sys_log (log_level);
CREATE INDEX idx_sys_log_service_name   ON sys_log (service_name);

-- ============================================================
-- HTTP 请求日志表模板（按天分表：sys_log_http_request_yyyyMMdd）
-- ============================================================
CREATE TABLE sys_log_http_request (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    log_id          BIGINT          NOT NULL,
    http_method     VARCHAR(16),
    uri             VARCHAR(1024),
    url             TEXT,
    query_string    TEXT,
    headers         TEXT,
    cookies         TEXT,
    request_body    TEXT,
    parameters      TEXT,
    content_type    VARCHAR(256),
    user_agent      TEXT,
    referer         TEXT,
    protocol        VARCHAR(16),
    created_at      TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP
);

COMMENT ON TABLE  sys_log_http_request              IS 'HTTP请求日志表模板（按天分表）';
COMMENT ON COLUMN sys_log_http_request.id           IS '主键ID';
COMMENT ON COLUMN sys_log_http_request.log_id       IS '关联 sys_log.id';
COMMENT ON COLUMN sys_log_http_request.http_method  IS 'HTTP方法';
COMMENT ON COLUMN sys_log_http_request.uri          IS '请求URI';
COMMENT ON COLUMN sys_log_http_request.url          IS '完整URL';
COMMENT ON COLUMN sys_log_http_request.query_string IS '查询参数';
COMMENT ON COLUMN sys_log_http_request.headers      IS '请求头（JSON，已脱敏）';
COMMENT ON COLUMN sys_log_http_request.cookies      IS 'Cookie（JSON，已脱敏）';
COMMENT ON COLUMN sys_log_http_request.request_body IS '请求体（已脱敏）';
COMMENT ON COLUMN sys_log_http_request.parameters   IS '请求参数（JSON）';
COMMENT ON COLUMN sys_log_http_request.content_type IS 'Content-Type';
COMMENT ON COLUMN sys_log_http_request.user_agent   IS 'User-Agent';
COMMENT ON COLUMN sys_log_http_request.referer      IS 'Referer';
COMMENT ON COLUMN sys_log_http_request.protocol     IS '协议版本（HTTP/1.1、HTTP/2）';
COMMENT ON COLUMN sys_log_http_request.created_at   IS '创建时间';

CREATE INDEX idx_sys_log_http_req_log_id ON sys_log_http_request (log_id);

-- ============================================================
-- HTTP 响应日志表模板（按天分表：sys_log_http_response_yyyyMMdd）
-- ============================================================
CREATE TABLE sys_log_http_response (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    log_id          BIGINT          NOT NULL,
    status_code     INT,
    headers         TEXT,
    response_body   TEXT,
    content_type    VARCHAR(256),
    content_length  BIGINT,
    error_code      VARCHAR(64),
    error_message   TEXT,
    created_at      TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP
);

COMMENT ON TABLE  sys_log_http_response                 IS 'HTTP响应日志表模板（按天分表）';
COMMENT ON COLUMN sys_log_http_response.id              IS '主键ID';
COMMENT ON COLUMN sys_log_http_response.log_id          IS '关联 sys_log.id';
COMMENT ON COLUMN sys_log_http_response.status_code     IS 'HTTP状态码';
COMMENT ON COLUMN sys_log_http_response.headers         IS '响应头（JSON）';
COMMENT ON COLUMN sys_log_http_response.response_body   IS '响应体（已脱敏）';
COMMENT ON COLUMN sys_log_http_response.content_type    IS 'Content-Type';
COMMENT ON COLUMN sys_log_http_response.content_length  IS '内容长度';
COMMENT ON COLUMN sys_log_http_response.error_code      IS '业务错误码';
COMMENT ON COLUMN sys_log_http_response.error_message   IS '业务错误消息';
COMMENT ON COLUMN sys_log_http_response.created_at      IS '创建时间';

CREATE INDEX idx_sys_log_http_resp_log_id ON sys_log_http_response (log_id);
```

#### 4.6.5 分表策略说明

| 项目 | 策略 |
|------|------|
| 分表粒度 | 按天，表名格式 `sys_log_yyyyMMdd` |
| 路由键 | traceId 前6位（`yyMMdd`），由 `LogTableRouter.resolveTableName()` 解析 |
| 建表时机 | 每天 23:55 由调度任务预建次日三张分表 |
| 写入路由 | `AsyncDbLogWriter.flush()` 按目标表分组批量写入 |
| 表结构同步 | `CREATE TABLE IF NOT EXISTS ... (LIKE base_table INCLUDING ALL)` 复制模板表全部结构 |
| 数据保留 | 默认保留30天，超期表由每日 02:00 调度任务执行 `DROP TABLE` |
| 无 traceId 兜底 | 路由到当天表（`sys_log_yyyyMMdd`） |

### 4.7 配置项设计

```yaml
hy:
  log:
    enabled: true                     # 是否启用日志模块

    format: json                      # 日志格式：json / text

    appender:
      buffer-size: 8192               # Disruptor 队列大小（2的幂次）

    output:
      console:
        enabled: true
      file:
        enabled: true
        path: /var/log/houyu
        max-size: 100MB
        max-history: 30
      db:
        enabled: false                # DB 后置处理默认关闭
        batch-size: 100               # 批量写入大小
        flush-interval: 5000          # 刷新间隔（毫秒）
        retention-days: 30            # 分表保留天数

    desensitize:
      enabled: true
      patterns:
        phone: "1[3-9]\\d{9}"
        email: "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"

    filter:
      min-level: INFO
      exclude-packages:
        - org.springframework
        - com.alibaba.druid

    sampler:
      enabled: false
      percentage: 10

    trace:
      enabled: true
      header-names:
        trace-id: X-Trace-Id
        span-id: X-Span-Id
        parent-span-id: X-Parent-Span-Id
        trace-flag: X-Trace-Flag
      redis-address: "redis://localhost:6379"   # 机器码注册 Redis 地址
      redis-database: 0

    replay:
      enabled: true
      capture-request: true
      capture-response: true
      max-body-size: 1048576
      exclude-paths:
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

通过 Spring Boot AutoConfiguration 自动装配，提供合理默认值，无需额外配置即可使用。如需覆盖默认行为，在 `application.yml` 中按第 4.7 节配置项调整。

## 6. 扩展机制

### 6.1 自定义格式化器

实现 `LogFormatter` 接口并注册为 Spring Bean，模块自动发现并使用：

```java
public interface LogFormatter {
    String format(HyLogEvent event);
}
```

### 6.2 自定义脱敏器

实现 `Desensitizer` 接口并注册为 Spring Bean，`DesensitizerManager` 自动注入：

```java
public interface Desensitizer {
    String desensitize(String value);
    boolean support(DesensitizeType type);
}
```

## 7. 性能优化

- **异步接管**：`HyCommonLogAppender` 使用 LMAX Disruptor，`tryNext()` 非阻塞，业务线程零等待
- **延迟格式化**：`prepareForDeferredProcessing()` 冻结 event 数据后交由消费线程格式化
- **批量 DB 写入**：`AsyncDbLogWriter` 内存缓冲 + 定时批量刷入，减少数据库 RTT
- **日志采样**：高并发环境可按百分比降低日志量
- **DB 旁路隔离**：DB 失败不影响文件输出主链路

## 8. 版本历史

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.0.0 | 2026-05-13 | 初始版本：编程式 Logback 接管、19位 traceId、JetCache+Redisson 机器码注册、PostgreSQL 日分表、API 重放支持 |
