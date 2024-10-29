下面是一个完整的示例，展示了如何使用 `@Around` 注解来创建一个切面，并在方法调用前后记录日志和计算执行时间。这个示例将包括一个简单的 Spring Boot 应用程序，其中包含服务层、控制器层以及一个 AOP 切面。

### 项目结构
```
src
├── main
│   ├── java
│   │   └── com
│   │       └── example
│   │           └── demo
│   │               ├── DemoApplication.java
│   │               ├── controller
│   │               │   └── GreetingController.java
│   │               ├── service
│   │               │   └── GreetingService.java
│   │               └── aspect
│   │                   └── LoggingAspect.java
│   └── resources
│       └── application.properties
└── test
    └── java
        └── com
            └── example
                └── demo
                    └── DemoApplicationTests.java
```

### 1. 添加依赖
首先，在 `pom.xml` 中添加必要的依赖：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 2. 创建服务类
创建一个简单的服务类 `GreetingService`：

```java
package com.example.demo.service;

import org.springframework.stereotype.Service;

@Service
public class GreetingService {
    public String greet(String name) {
        // 模拟一些处理逻辑
        try {
            Thread.sleep(1000); // 模拟耗时操作
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return "Hello, " + name + "!";
    }
}
```

### 3. 创建控制器
创建一个控制器 `GreetingController` 来暴露 REST API：

```java
package com.example.demo.controller;

import com.example.demo.service.GreetingService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class GreetingController {

    @Autowired
    private GreetingService greetingService;

    @GetMapping("/greet")
    public String greet(@RequestParam(value = "name", defaultValue = "World") String name) {
        return greetingService.greet(name);
    }
}
```

### 4. 创建 AOP 切面
创建一个 AOP 切面 `LoggingAspect`，并在方法调用前后记录日志和计算执行时间：

```java
package com.example.demo.aspect;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class LoggingAspect {

    private static final Logger logger = LoggerFactory.getLogger(LoggingAspect.class);

    @Around("execution(* com.example.demo.service.*.*(..))")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();

        // 获取方法签名
        String methodName = joinPoint.getSignature().getName();
        logger.info("开始执行方法: " + methodName);

        try {
            // 执行目标方法
            Object result = joinPoint.proceed();
            return result;
        } finally {
            long executionTime = System.currentTimeMillis() - start;
            logger.info("方法执行结束: " + methodName + ", 耗时: " + executionTime + "ms");
        }
    }
}
```

### 5. 创建主应用程序
创建主应用程序 `DemoApplication`：

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

### 6. 运行应用程序
运行 `DemoApplication` 启动应用程序。然后你可以通过浏览器或 Postman 等工具访问 `http://localhost:8080/greet?name=John` 来测试 API。

### 7. 查看日志
在控制台中，你应该会看到类似如下的日志输出：

```
INFO  [com.example.demo.aspect.LoggingAspect] - 开始执行方法: greet
INFO  [com.example.demo.aspect.LoggingAspect] - 方法执行结束: greet, 耗时: 1002ms
```

