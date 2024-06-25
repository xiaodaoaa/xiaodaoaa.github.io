在 Java 中，`@Scheduled` 是一个注解，用于在 Spring 框架中配置定时任务。它可以让你在特定的时间间隔或特定的时间点执行某个方法。`@Scheduled` 注解通常和 `@EnableScheduling` 注解一起使用，以启用 Spring 的调度功能。

### 1. 基本使用

#### 1.1 添加依赖

确保你的 Spring 项目中包含以下依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

#### 1.2 启用调度功能

在你的配置类或启动类中添加 `@EnableScheduling` 注解：

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

#### 1.3 使用 `@Scheduled` 注解

创建一个带有 `@Scheduled` 注解的方法：

```java
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class ScheduledTasks {

    @Scheduled(fixedRate = 5000)
    public void reportCurrentTime() {
        System.out.println("Current Time: " + System.currentTimeMillis());
    }
}
```

### 2. `@Scheduled` 注解的参数

`@Scheduled` 注解提供了多种配置参数，用于指定任务的执行时间或间隔：

#### 2.1 `fixedRate`

`fixedRate` 表示任务以固定的频率执行，单位为毫秒。上一次任务开始执行的时间和下一次任务开始执行的时间之间的间隔是固定的。

```java
@Scheduled(fixedRate = 5000)
public void fixedRateTask() {
    System.out.println("Fixed rate task - " + System.currentTimeMillis());
}
```

#### 2.2 `fixedDelay`

`fixedDelay` 表示任务在前一个任务完成之后延迟指定时间后再执行，单位为毫秒。上一次任务完成的时间和下一次任务开始执行的时间之间的间隔是固定的。

```java
@Scheduled(fixedDelay = 5000)
public void fixedDelayTask() {
    System.out.println("Fixed delay task - " + System.currentTimeMillis());
}
```

#### 2.3 `initialDelay`

`initialDelay` 表示任务在第一次执行前的延迟时间，单位为毫秒。通常和 `fixedRate` 或 `fixedDelay` 一起使用。

```java
@Scheduled(fixedRate = 5000, initialDelay = 10000)
public void initialDelayTask() {
    System.out.println("Initial delay task - " + System.currentTimeMillis());
}
```

#### 2.4 `cron`

`cron` 表示任务根据 Cron 表达式执行。Cron 表达式非常强大，可以精确到秒级别。

```java
@Scheduled(cron = "0 * * * * *")
public void cronTask() {
    System.out.println("Cron task - " + System.currentTimeMillis());
}
```

Cron 表达式的格式为：`秒 分 小时 日 月 星期 [年]`。例如：

- `"0 0 12 * * ?" `: 每天中午12点触发
- `"0 15 10 * * ?"`: 每天上午10:15触发
- `"0 0/5 14 * * ?"`: 每天下午2点到2:55之间每5分钟触发

### 3. 完整示例

下面是一个完整的 Spring Boot 应用示例，其中包含了 `@Scheduled` 注解的各种用法：

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@SpringBootApplication
@EnableScheduling
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}

@Component
class ScheduledTasks {

    @Scheduled(fixedRate = 5000)
    public void fixedRateTask() {
        System.out.println("Fixed rate task - " + System.currentTimeMillis());
    }

    @Scheduled(fixedDelay = 5000)
    public void fixedDelayTask() {
        System.out.println("Fixed delay task - " + System.currentTimeMillis());
    }

    @Scheduled(fixedRate = 5000, initialDelay = 10000)
    public void initialDelayTask() {
        System.out.println("Initial delay task - " + System.currentTimeMillis());
    }

    @Scheduled(cron = "0 * * * * *")
    public void cronTask() {
        System.out.println("Cron task - " + System.currentTimeMillis());
    }
}
```

### 结论

通过 `@Scheduled` 注解，Spring 提供了一种简洁的方式来配置定时任务。你可以根据需要选择不同的参数来控制任务的执行时间和频率。注意，`@Scheduled` 注解的方法不能有参数，返回值应为 `void`。此外，为了确保任务执行的稳定性，建议在实际应用中处理可能出现的异常情况。