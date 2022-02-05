---
title: 理解 Spring Boot 日志框架
date: 2021-12-13 10:54:20
tags:
- Java
- Spring Boot
---

## 1. Spring Boot 日志框架

Spring Boot 日志框架通过 `org.springframework.boot:spring-boot-starter-logging` Starter 启用，由于常用的 Spring Boot Starter 已经使用了该 Starter，所以通常无需在自己的代码中直接引用。

Spring Boot 使用 [SLF4J](http://www.slf4j.org/) 为应用开发提供日志接口，而 Spring Boot 内部则采用了 [Commons Logging](https://commons.apache.org/proper/commons-logging/) 作为日志接口，这两个组件都是负责将实际日志操作转接到对应的底层日志实现。日志底层实现可以支持 [Logback](https://logback.qos.ch/)、[Log4j](https://logging.apache.org/log4j/2.x/) 和 [Java Utils Logging](https://docs.oracle.com/javase/8/docs/api/java/util/logging/package-summary.html) 这三种日志系统，默认使用 Logback 系统。

在 Spring Boot 应用程序启动后，日志框架会通过 `org.springframework.boot.context.logging.LoggingApplicationListener` 侦听器处理启动阶段的事件，逐步完成日志系统的配置。

{% asset_img components.jpg %}

<!-- more -->

## 2. 配置和记录日志

### 2.1. 记录日志

代码中通过 SLF4J 日志接口可以很方便的记录日志，并自动采用 Spring Boot 日志框架的配置。

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

class MockServiceImpl implements MockService {

    private Logger logger = LoggerFactory.getLogger(MockServiceImpl.class);

    @Override
    public void doSomething() {
        logger.info("do something");
    }
}
```

### 2.2. 无配置

Spring Boot 应用程序默认已经启用了日志框架，无需单独进行任何配置就可向控制台输出日志信息，日志输出内容参见如下截图。

{% asset_img screen-snapshot-1.png %}

### 2.3. YAML 配置文件

通常在 _application.yml_ 配置文件的 `logging` 配置项下进行 Spring Boot 日志框架的配置。

#### 2.3.1. 配置日志级别

Spring Boot 日志框架使用 SLF4J 提供日志接口，支持 ERROR、WARN、INFO、DEBUG、TRACE 五个级别，可在 `logging.level` 配置项下配置输出的日志级别，默认全局输出 INFO 级别。

```yaml
logging:
  level:
    root: trace
    name.xupeng.spring.boot.logging.yamlconfig.UseLogClass: warn
```

上述代码设置 `logging.level.root` 全局日志级别为 TRACE，该全局配置同样会影响到 Spring Boot 的日志输出。`name.xupeng.spring.boot.logging.yamlconfig.UseLogClass` 配置了对应类型的日志级别为 WARN，该配置仅会影响到 `UseLogClass` 类型的日志输出。另外，也可以针对包名称进行日志级别的配置。

#### 2.3.2. 配置 Spring Boot 日志级别

如果未对日志框架进行 `logging.level.root` 全局配置，则可以使用 `debug` 和 `trace` 单独配置 Spring Boot 的日志输出级别，这两个配置项的默认值均为 `false`。以下配置使应用程序输出 INFO 级别的日志，应用程序的 UseLogClass 类型输出 WARN 级别的日志，而 Spring Boot 输出 DEBUG 级别的日志。

```yaml
logging:
  level:
    name.xupeng.spring.boot.logging.yamlconfig.UseLogClass: warn
debug: true
```

#### 2.3.3. 配置日志文件输出

默认情况下，Spring Boot 只记录日志到控制台。如果需要同时将日志写入到日志文件，可以设置 `logging.file.name` 或 `logging.file.path` 配置项，分别指定将日志写入指定文件或指定目录。日志文件在达到 10MB 时会自动轮换，如果使用的 Logback 日志系统，则可以通过 `logging.logback.rollingpolicy` 配置轮换，其他日志系统则需要单独日志系统的配置文件。

### 2.4. 日志系统配置文件

Spring Boot 为日志系统提供了一套默认的配置方案。如果需要定制日志系统，Spring Boot 也可以根据当前启用的日志系统（默认为 Logback），加载日志系统对应的配置文件进行个性化定制。

以下是 Spring Boot 支持的日志系统对应的配置文件，建议使用后缀为 _-spring_ 的变体配置文件，这样可以使 Spring Boot 完全控制日志系统的初始化。

| 日志系统 | 配置文件 |
| --------- | -------- |
| Logback | logback-spring.xml, logback.xml, logback-spring.groovy, logback.groovy |
| Log4j | log4j2-spring.xml, log4j2.xml |
| JDK（Java Util Logging） | logging.properties |

另外，Java Util Logging 存在已知的类加载问题，而这些问题会导致从 _可执行 JAR_ 运行时出现问题，应当避免在 _可执行 JAR_ 中使用。

### 2.5. 测试环境配置文件

由于 Spring Boot 日志框架的配置依赖于 Spring Boot 应用程序的启动过程，而测试环境在运行时并没有启动任何 Spring Boot 应用程序，因此前述的配置方式并不完成适用。

测试环境使用的日志系统应当与 Spring Boot 应用程序使用的日志系统一致，测试环境的日志系统配置则需要按照启用的日志系统的本来方式进行配置。

默认情况下，测试环境下使用的 Logback 日志系统，可以在 _src/test/resources_ 目录下创建 _logback-test.xml_ 测试环境日志配置文件。以下配置内容参考 Spring Boot 的默认日志配置方案进行设置。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="false">
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %highlight(%5level) --- [%10.10thread] %boldCyan(%-40.40logger{40}) : %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="org.apache.commons.beanutils" level="INFO" />
    <logger name="org.apache.poi" level="INFO" />

    <root level="DEBUG">
        <appender-ref ref="CONSOLE" />
    </root>
</configuration>
```

## 3. 使用不同的日志系统

### 3.1. 启用 Log4j 日志系统

Spring Boot 日志框架通过 `org.springframework.boot:spring-boot-starter-log4j2` Starter 启用 Log4j 日志系统，该 Starter 与 `org.springframework.boot:spring-boot-starter-logging` 并不兼容，因此需要很小心的从其他依赖关系中排除对 `org.springframework.boot:spring-boot-starter-logging` 的引用。

### 3.2. 启用 Java Util Logging 日志系统

由于 Java Util Logging 日志系统由 JDK 自带提供，要启用他必须采用强制设定的方案。通过在启动 Spring Boot 应用程序时提供 `org.springframework.boot.logging.LoggingSystem` 这个系统属性，并设置其值为 `org.springframework.boot.logging.java.JavaLoggingSystem`，让 Spring Boot 强制使用 Java Util Logging 日志系统。

运行 Spring Boot 应用程序的命令行看起来与下类似。

```bash
java -Dorg.springframework.boot.logging.LoggingSystem=org.springframework.boot.logging.java.JavaLoggingSystem -jar executable.jar
```

## 4. 日志框架的实现原理

Spring Boot 日志框架的核心目标，是在 Spring 框架的基础上，让开发者在使用 SLF4J 进行日志记录时，还可以根据配置自由切换底层日志系统，并且保证日志框架与 Spring 框架的融合。

在 [使用不同的日志系统](#使用不同的日志系统) 里我们看到，日志框架通过不同的 Starter 或配置来启用不同的日志系统。如果我们不考虑定制日志方案，那么启用不同的日志系统并不会让我们在 Spring Boot 的配置发生变化。

+ Logback，使用 `spring-boot-starter-logging` Starter 启用
+ Log4j，使用 `spring-boot-starter-log4j2` Starter 启用
+ Java Util Logging，设置系统属性 `org.springframework.boot.logging.LoggingSystem` 强制启用

Spring Boot 日志框架在启用日志系统时，优先使用 `org.springframework.boot.logging.LoggingSystem` 系统属性指定的日志系统（同时需要正确配置依赖关系），如果没有指定日志系统，则按顺序检查类路径是否存在以下日志系统，并启用最先找到的那个。

+ Logback
+ Log4j
+ Java Util Logging

### 4.1. 基于 Logback 的日志框架

下图展示了 `spring-boot-starter-logging` Starter 的依赖关系。

{% asset_img screen-snapshot-2.png %}

其中，`jul-to-slf4j` 将 Java Util Logging 日志系统转接到 SLF4J，`log4j-to-slf4j` 将 Log4j 日志系统转接到 SLF4J，这样任何使用 Log4j 或 Java Util Logging 日志系统的组件都将通过 SLF4J 转交 Logback 处理。

### 4.2. 基于 Log4j 的日志框架

下图展示了 `spring-boot-starter-log4j2` Starter 的依赖关系。

{% asset_img screen-snapshot-3.png %}

其中，`log4j-jul` 提供了 `java.util.logging.LogManager` 的使用 Log4j 的自定义实现，将 Java Util Logging 转接到 Log4j。

注意，这里没有 Logback 转接到 Log4j 的组件，再加上之前说到的日志系统检查顺序，这就是为什么需要在启用 Log4j 日志系统时排除 `spring-boot-starter-logging` Starter 的原因了。

### 4.3. 基于 Java Util Logging 的日志框架

Java Util Loggin 由 JDK 提供，当启用 JUL 时，Spring Boot 会进行适当配置，以使所有日志输出该日志系统。

## 5. 参考资料

[Spring Boot 2.6.1](https://docs.spring.io/spring-boot/docs/2.6.1/reference/html/)
[spring-boot-logging-demo](https://github.com/athrunsaga/demo-projects/tree/master/spring-demos/spring-boot-logging-demo)
