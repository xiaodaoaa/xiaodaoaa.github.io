如果你不启用 `@Async` 注解来执行异步任务，你可以直接使用 `ThreadPoolTaskExecutor` 来提交任务。在这种情况下，你需要显式地调用 `ThreadPoolTaskExecutor` 的 `execute` 或 `submit` 方法来执行任务。以下是一个详细的示例：

### 1. 添加依赖

确保在你的 `pom.xml` 中添加了 Spring Context 的依赖：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>5.3.10</version>
    </dependency>
</dependencies>
```

### 2. 配置类

我们需要配置一个 `ThreadPoolTaskExecutor`。这个类负责创建和管理线程池。

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
public class TaskExecutorConfig {

    @Bean(name = "taskExecutor")
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);  // 核心线程数
        executor.setMaxPoolSize(10);  // 最大线程数
        executor.setQueueCapacity(25);  // 队列容量
        executor.setThreadNamePrefix("MyExecutor-");  // 线程名前缀
        executor.initialize();  // 初始化
        return executor;
    }
}
```

### 3. 使用 `ThreadPoolTaskExecutor` 执行任务

在服务类中使用 `ThreadPoolTaskExecutor` 来提交任务，而不是使用 `@Async` 注解。

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.stereotype.Service;

@Service
public class TaskService {

    @Autowired
    private ThreadPoolTaskExecutor taskExecutor;

    public void executeTask(int i) {
        taskExecutor.execute(() -> {
            System.out.println("Task " + i + " is executed by " + Thread.currentThread().getName());
        });
    }
}
```

#### 详解：

- `taskExecutor.execute(Runnable task)`: 使用 `ThreadPoolTaskExecutor` 提交一个 `Runnable` 任务。

### 4. 启动类

主类无需任何特殊注解，因为我们不使用 `@Async` 功能。

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 5. 运行任务

编写一个简单的测试类来验证 `ThreadPoolTaskExecutor` 的功能：

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

@Component
public class TaskRunner implements CommandLineRunner {

    @Autowired
    private TaskService taskService;

    @Override
    public void run(String... args) throws Exception {
        for (int i = 0; i < 10; i++) {
            taskService.executeTask(i);
        }
    }
}
```

#### 详解：

- `CommandLineRunner`: 一个简单的 Spring Boot 接口，用于在应用启动后执行一些代码。
- `taskService.executeTask(i)`: 调用 `executeTask` 方法，该方法将使用 `ThreadPoolTaskExecutor` 来执行任务。

### 6. 运行结果

运行这个 Spring Boot 应用程序，你会看到控制台输出类似以下内容：

```
Task 0 is executed by MyExecutor-1
Task 1 is executed by MyExecutor-2
Task 2 is executed by MyExecutor-3
Task 3 is executed by MyExecutor-4
Task 4 is executed by MyExecutor-5
Task 5 is executed by MyExecutor-1
Task 6 is executed by MyExecutor-2
Task 7 is executed by MyExecutor-3
Task 8 is executed by MyExecutor-4
Task 9 is executed by MyExecutor-5
```

每个任务都被不同的线程执行，说明 `ThreadPoolTaskExecutor` 正在正常工作。

### 总结

通过这个例子，我们展示了如何直接使用 `ThreadPoolTaskExecutor` 来管理和执行任务，而无需使用 `@Async` 注解。这种方式可以在不启用 Spring 异步支持的情况下实现多线程任务的执行。