# hy-common-log 架构设计文档

## 1. 模块概述

### 1.1 模块定位
- **模块名称**：hy-common-log
- **父模块**：hy-common
- **模块类型**：后端公共日志模块
- **核心职责**：接管 logback 记录的所有日志并进行格式化处理

### 1.2 设计目标
- 统一日志格式规范
- 提供可扩展的日志处理机制
- 支持多种日志输出渠道
- 简化各业务模块的日志配置
- 提供日志脱敏、过滤等增强功能

## 2. 架构设计

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        业务应用模块                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │  hy-user │  │ hy-order │  │ hy-goods│  │  ...     │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
└───────┼─────────────┼─────────────┼─────────────┼─────────────┘
        │             │             │             │
        ▼             ▼             ▼             ▼
┌─────────────────────────────────────────────────────────────────┐
│                        hy-common-log                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Logback 接管层                          │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │   │
│  │  │   日志监听器  │  │   上下文传递  │  │   MDC 管理    │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              ↓                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    日志格式化层                            │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │   │
│  │  │  JSON格式化  │  │  文本格式化  │  │  自定义格式  │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              ↓                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    日志增强处理层                          │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │   │
│  │  │   脱敏处理   │  │   过滤规则   │  │   采样策略   │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              ↓                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    日志输出层                              │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │   │
│  │  │   控制台输出 │  │    文件输出   │  │  异步输出    │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件分层

| 层级 | 组件名称 | 职责说明 |
|------|---------|---------|
| 接管层 | LogbackListener | 监听 logback 日志事件，接管所有日志输出 |
| 接管层 | LogContextHolder | 管理日志上下文（MDC），支持链路追踪 |
| 格式化层 | LogFormatter | 日志格式化抽象接口 |
| 格式化层 | JsonLogFormatter | JSON 格式日志实现 |
| 格式化层 | TextLogFormatter | 文本格式日志实现 |
| 增强层 | LogDesensitizer | 日志脱敏处理器 |
| 增强层 | LogFilter | 日志过滤器（按级别、关键词等） |
| 输出层 | LogAppender | 日志输出器抽象接口 |
| 输出层 | ConsoleAppender | 控制台输出实现 |
| 输出层 | FileAppender | 文件输出实现 |

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
│   │   │       ├── formatter/           # 格式化层
│   │   │       │   ├── LogFormatter.java
│   │   │       │   ├── JsonLogFormatter.java
│   │   │       │   └── TextLogFormatter.java
│   │   │       ├── desensitizer/        # 脱敏层
│   │   │       │   ├── Desensitizer.java
│   │   │       │   ├── PhoneDesensitizer.java
│   │   │       │   ├── EmailDesensitizer.java
│   │   │       │   └── DesensitizerManager.java
│   │   │       ├── filter/              # 过滤层
│   │   │       │   ├── LogFilter.java
│   │   │       │   ├── LevelLogFilter.java
│   │   │       │   └── KeywordLogFilter.java
│   │   │       ├── appender/            # 输出层
│   │   │       │   ├── LogAppender.java
│   │   │       │   ├── ConsoleLogAppender.java
│   │   │       │   └── FileLogAppender.java
│   │   │       ├── context/             # 上下文管理
│   │   │       │   └── LogContextHolder.java
│   │   │       ├── listener/            # 日志监听器
│   │   │       │   └── LogbackLogListener.java
│   │   │       ├── constant/            # 常量定义
│   │   │       │   └── LogConstants.java
│   │   │       └── util/                # 工具类
│   │   │           └── LogUtils.java
│   │   └── resources/
│   │       └── META-INF/
│   │           └── spring/
│   │               └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
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
    <description>HouYu Common Log Module - 公共日志模块，统一日志格式和输出</description>

    <dependencies>
        <!-- Spring Boot Starter -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <!-- Logback (Spring Boot 默认包含) -->
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
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

### 4.1 日志格式规范

#### 4.1.1 JSON 格式（生产环境推荐）

```json
{
  "timestamp": "2026-05-13T10:30:00.000+08:00",
  "level": "INFO",
  "traceId": "abc123def456",
  "spanId": "span789",
  "serviceName": "hy-user",
  "className": "com.houyu.user.service.UserService",
  "methodName": "getUserById",
  "message": "查询用户信息成功",
  "userId": 12345,
  "duration": 150,
  "exception": null,
  "threadName": "http-nio-8080-exec-1",
  "ip": "192.168.1.100"
}
```

#### 4.1.2 文本格式（开发环境推荐）

```
[2026-05-13 10:30:00.000] [INFO] [traceId=abc123def456] [service=hy-user] [thread=http-nio-8080-exec-1]
  com.houyu.user.service.UserService.getUserById(123) - 查询用户信息成功 | userId=12345, duration=150ms
```

### 4.2 日志脱敏设计

#### 4.2.1 支持的脱敏类型

| 类型 | 示例 | 脱敏后 |
|------|------|--------|
| 手机号 | 13812345678 | 138****5678 |
| 邮箱 | test@example.com | te**@example.com |
| 身份证 | 110101199001011234 | 110***********1234 |
| 银行卡 | 6222021234567890123 | 6222***********0123 |
| 密码 | password123 | ****** |

#### 4.2.2 脱敏注解

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LogDesensitize {
    DesensitizeType type();
}
```

### 4.3 链路追踪支持

- 集成 MDC（Mapped Diagnostic Context）
- 自动传递 traceId、spanId
- 支持微服务间链路信息传递

### 4.4 配置项设计

```yaml
hy:
  log:
    enabled: true                    # 是否启用日志模块
    format: json                     # 日志格式: json/text
    appenders:                       # 输出目标
      - console
      - file
    desensitize:
      enabled: true                  # 是否启用脱敏
      patterns:                      # 自定义脱敏正则
        - phone: "1[3-9]\\d{9}"
    filter:
      min-level: INFO               # 最低日志级别
      exclude-packages:             # 排除的包
        - org.springframework.*
    file:
      path: /var/log/houyu          # 日志文件路径
      max-size: 100MB               # 单个文件最大大小
      max-history: 30               # 保留天数
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
    String format(LogEvent event);
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

- 异步日志输出（使用 Disruptor 高性能队列）
- 日志采样（降低高并发下的日志量）
- 延迟格式化（避免不必要的字符串拼接）
- 批量写入文件

## 8. 版本历史

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.0.0 | 2026-05-13 | 初始版本，基础日志格式化和脱敏功能 |
