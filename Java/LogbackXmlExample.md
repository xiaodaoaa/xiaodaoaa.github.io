```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <property name="component.id" value="mhttpserver"/>
    <!-- 定义日志输出格式 -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>DEBUG</level>
        </filter>
        <encoder>
            <pattern>%d{yyyy-MM-dd'T'HH:mm:ss.SSSXXX} %highlight(%-5level) ${component.id} [%thread] [%cyan(%logger{50}):%line] %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>
    <!-- 定义文件日志输出 -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/${component.id}.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <!-- 设置滚动文件的索引范围 -->
            <MinIndex>1</MinIndex>
            <MaxIndex>10</MaxIndex>
            <!-- 设置滚动文件的名称模式并压缩 -->
            <FileNamePattern>logs/${component.id}.%i.log.zip</FileNamePattern>
        </rollingPolicy>
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <!-- 文件大小设置为100M-->
            <MaxFileSize>100MB</MaxFileSize>
        </triggeringPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd'T'HH:mm:ss.SSSXXX} %level ${component.id} [%thread] [%logger{50}:%line] - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>
    <!-- 针对特定包或类设置日志级别 -->
    <logger name="org.springframework" level="warn"/>
    <!--<logger name="com.example.mhttpserver" additivity="false" level="debug">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </logger>-->
    <!-- 设置日志级别 -->
    <root level="info">
        <!-- 输出到控制台-->
        <appender-ref ref="CONSOLE"/>
        <!-- 输出到文件 -->
        <appender-ref ref="FILE"/>
    </root>
</configuration>

```

