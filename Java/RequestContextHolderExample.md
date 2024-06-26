在 Spring Boot 中，`RequestContextHolder` 是一个非常有用的类，它可以让你在非控制器类中访问当前的 HTTP 请求对象。这在处理与当前请求相关的逻辑时特别有用，例如在服务类或其他非控制器组件中获取请求信息。

以下是对 `RequestContextHolder` 的详细解释以及如何在 Spring Boot 项目中使用它的示例。

### 1. `RequestContextHolder` 简介

`RequestContextHolder` 是 Spring 框架的一部分，它提供了一种线程安全的方式来访问与当前线程绑定的 `RequestAttributes`。通过 `RequestAttributes`，你可以获取当前请求的 `HttpServletRequest` 对象。

`RequestContextHolder` 的主要方法有：

- `getRequestAttributes()`：获取当前线程绑定的 `RequestAttributes` 对象。
- `setRequestAttributes(RequestAttributes attributes)`：将 `RequestAttributes` 对象绑定到当前线程。
- `resetRequestAttributes()`：移除当前线程绑定的 `RequestAttributes` 对象。

### 2. 在 Spring Boot 中使用 `RequestContextHolder`

下面是一个示例，展示如何在 Spring Boot 项目中使用 `RequestContextHolder` 来获取当前请求的 `HttpServletRequest` 对象，并从中提取请求信息。

#### 依赖

确保你的项目中包含 Spring Web 依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

#### 控制器

创建一个简单的控制器来处理请求：

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class MyController {

    @GetMapping("/request-info")
    public String getRequestInfo() {
        return MyService.getRequestInfo();
    }
}
```

#### 服务类

在服务类中使用 `RequestContextHolder` 获取当前请求的 `HttpServletRequest` 对象：

```java
import org.springframework.stereotype.Service;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import javax.servlet.http.HttpServletRequest;

@Service
public class MyService {

    public static String getRequestInfo() {
        HttpServletRequest request = getCurrentHttpRequest();
        if (request == null) {
            return "No current request";
        }

        String clientIp = request.getRemoteAddr();
        String protocol = request.getProtocol();
        String method = request.getMethod();
        String domain = request.getServerName();

        return String.format("Domain: %s, Client IP: %s, Protocol: %s, Request Method: %s", domain, clientIp, protocol, method);
    }

    private static HttpServletRequest getCurrentHttpRequest() {
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (attributes != null) {
            return attributes.getRequest();
        }
        return null;
    }
}
```

### 示例解释

1. **控制器**：
   - 在控制器中，我们定义了一个 `GET` 请求的处理方法，该方法调用 `MyService.getRequestInfo()` 获取请求信息。

2. **服务类**：
   - 在服务类 `MyService` 中，我们定义了一个静态方法 `getRequestInfo()`，该方法使用 `RequestContextHolder` 获取当前请求的 `HttpServletRequest` 对象。
   - `getCurrentHttpRequest()` 方法通过 `RequestContextHolder.getRequestAttributes()` 获取当前线程的 `RequestAttributes`，并从中提取 `HttpServletRequest`。
   - 然后，我们从 `HttpServletRequest` 对象中获取客户端 IP、协议、请求方法和域名等信息，并返回这些信息的格式化字符串。

### 运行应用程序

启动你的 Spring Boot 应用程序，然后访问 `http://localhost:8080/request-info`，你将看到类似如下的响应：

```
Domain: localhost, Client IP: 127.0.0.1, Protocol: HTTP/1.1, Request Method: GET
```

### 结论

`RequestContextHolder` 提供了一种方便的方法，可以在非控制器类中访问与当前请求相关的信息。通过使用 `RequestContextHolder`，你可以在服务层或其他组件中获取 `HttpServletRequest` 对象，从而访问请求的详细信息。这在复杂的应用程序中尤其有用，可以帮助你保持代码的清晰和模块化。