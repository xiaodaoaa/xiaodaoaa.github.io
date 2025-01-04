[Hutool](https://hutool.cn/) 是一个 Java 工具库，提供了丰富的工具类和方法，可以简化开发工作。Hutool 中的 `HttpUtil` 和 `ServletUtil` 类可以帮助我们更方便地获取客户端的 IP 地址。

以下是使用 Hutool 获取客户端 IP 地址的示例：

---

### 1. 添加 Hutool 依赖

首先，在你的 `pom.xml` 中添加 Hutool 的依赖：

```xml
<dependency>
    <groupId>cn.hutool</groupId>
    <artifactId>hutool-all</artifactId>
    <version>5.8.11</version> <!-- 请使用最新版本 -->
</dependency>
```

---

### 2. 使用 `ServletUtil` 获取客户端 IP

Hutool 的 `ServletUtil` 类提供了 `getClientIP` 方法，可以自动处理 `X-Forwarded-For` 头信息，并返回客户端的真实 IP 地址。

#### 示例代码：

```java
import cn.hutool.extra.servlet.ServletUtil;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.servlet.http.HttpServletRequest;

@RestController
public class IpController {

    @GetMapping("/get-ip")
    public String getClientIp(HttpServletRequest request) {
        // 使用 Hutool 的 ServletUtil 获取客户端 IP
        String clientIp = ServletUtil.getClientIP(request);
        return "Client IP: " + clientIp;
    }
}
```

#### 说明：
- `ServletUtil.getClientIP(request)` 方法会自动处理以下情况：
  - 如果请求经过代理服务器，会从 `X-Forwarded-For` 头信息中提取客户端的真实 IP。
  - 如果 `X-Forwarded-For` 不存在，则回退到 `request.getRemoteAddr()`。

---

### 3. 处理多级代理的情况

如果请求经过多级代理，`X-Forwarded-For` 头信息可能包含多个 IP 地址（例如：`client_ip, proxy1_ip, proxy2_ip`）。`ServletUtil.getClientIP` 默认会返回第一个 IP 地址，即客户端的真实 IP。

如果你需要自定义处理逻辑，可以手动解析 `X-Forwarded-For`：

```java
import cn.hutool.extra.servlet.ServletUtil;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.servlet.http.HttpServletRequest;

@RestController
public class IpController {

    @GetMapping("/get-ip")
    public String getClientIp(HttpServletRequest request) {
        // 获取 X-Forwarded-For 头信息
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        String clientIp;

        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            // 取第一个 IP（客户端真实 IP）
            clientIp = xForwardedFor.split(",")[0].trim();
        } else {
            // 回退到 Hutool 的默认方法
            clientIp = ServletUtil.getClientIP(request);
        }

        return "Client IP: " + clientIp;
    }
}
```

---

### 4. 使用 `HttpUtil` 获取 IP（非 Servlet 环境）

如果你不在 Servlet 环境（例如在普通 Java 应用中），可以使用 `HttpUtil` 获取本机 IP 地址：

```java
import cn.hutool.http.HttpUtil;

public class IpExample {
    public static void main(String[] args) {
        // 获取本机 IP 地址
        String localIp = HttpUtil.getLocalhostStr();
        System.out.println("Local IP: " + localIp);
    }
}
```

---

### 5. 注意事项

- **安全性**：
  - `X-Forwarded-For` 头信息可以被客户端伪造，因此在生产环境中需要确保代理服务器或负载均衡器正确设置了该头信息。
  - 如果请求未经过代理，直接使用 `request.getRemoteAddr()` 即可。

- **IPv6 支持**：
  - Hutool 的 `ServletUtil.getClientIP` 方法支持 IPv6 地址。

---

### 6. 总结

使用 Hutool 的 `ServletUtil.getClientIP` 方法可以非常方便地获取客户端的真实 IP 地址，尤其是在请求经过代理服务器或负载均衡器的情况下。Hutool 的封装大大简化了代码，避免了手动解析 `X-Forwarded-For` 的复杂性。

如果你还没有使用 Hutool，强烈推荐将其引入你的项目中，它的工具类和方法可以显著提高开发效率！如果有其他问题，欢迎随时提问。