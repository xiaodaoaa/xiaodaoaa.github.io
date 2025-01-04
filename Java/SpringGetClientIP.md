在 Java Spring Boot 中，获取 HTTP 请求客户端的 IP 地址是一个常见的需求。由于客户端请求可能会经过代理服务器、负载均衡器等中间节点，因此需要正确处理 `X-Forwarded-For` 等 HTTP 头信息。

以下是几种获取客户端 IP 地址的方法：

---

### 方法 1：使用 `HttpServletRequest` 直接获取

在 Spring Boot 中，可以通过 `HttpServletRequest` 对象获取客户端的 IP 地址。

```java
import javax.servlet.http.HttpServletRequest;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class IpController {

    @GetMapping("/get-ip")
    public String getClientIp(HttpServletRequest request) {
        // 获取客户端 IP 地址
        String ipAddress = request.getRemoteAddr();
        return "Client IP: " + ipAddress;
    }
}
```

#### 说明：
- `request.getRemoteAddr()` 直接返回客户端的 IP 地址。
- 如果请求经过代理服务器或负载均衡器，`getRemoteAddr()` 可能返回的是代理服务器的 IP，而不是客户端的真实 IP。

---

### 方法 2：处理 `X-Forwarded-For` 头信息

如果请求经过代理服务器或负载均衡器，客户端的真实 IP 通常会存储在 `X-Forwarded-For` 头信息中。`X-Forwarded-For` 是一个逗号分隔的 IP 地址列表，第一个 IP 是客户端的真实 IP。

```java
import javax.servlet.http.HttpServletRequest;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class IpController {

    @GetMapping("/get-ip")
    public String getClientIp(HttpServletRequest request) {
        // 从 X-Forwarded-For 头信息中获取客户端 IP
        String ipAddress = request.getHeader("X-Forwarded-For");
        if (ipAddress == null || ipAddress.isEmpty() || "unknown".equalsIgnoreCase(ipAddress)) {
            ipAddress = request.getRemoteAddr();
        } else {
            // 取第一个 IP（客户端真实 IP）
            ipAddress = ipAddress.split(",")[0].trim();
        }
        return "Client IP: " + ipAddress;
    }
}
```

#### 说明：
- `X-Forwarded-For` 头信息可能包含多个 IP 地址（例如：`client_ip, proxy1_ip, proxy2_ip`）。
- 如果 `X-Forwarded-For` 不存在，则回退到 `request.getRemoteAddr()`。

---

### 方法 3：使用 Spring 的工具类

Spring 提供了 `RequestContextHolder` 工具类，可以在任何地方获取当前请求的 `HttpServletRequest` 对象。

```java
import javax.servlet.http.HttpServletRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

@Component
public class IpService {

    public String getClientIp() {
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();
        String ipAddress = request.getHeader("X-Forwarded-For");
        if (ipAddress == null || ipAddress.isEmpty() || "unknown".equalsIgnoreCase(ipAddress)) {
            ipAddress = request.getRemoteAddr();
        } else {
            ipAddress = ipAddress.split(",")[0].trim();
        }
        return ipAddress;
    }
}
```

#### 使用示例：
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class IpController {

    @Autowired
    private IpService ipService;

    @GetMapping("/get-ip")
    public String getClientIp() {
        String ipAddress = ipService.getClientIp();
        return "Client IP: " + ipAddress;
    }
}
```

---

### 方法 4：使用 Spring Cloud Gateway 或 Zuul 的过滤器

如果你的应用使用了 Spring Cloud Gateway 或 Zuul 作为网关，可以在网关层面获取客户端的 IP 地址，并将其传递到下游服务。

#### 示例（Spring Cloud Gateway）：
```java
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Component
public class ClientIpFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String clientIp = request.getHeaders().getFirst("X-Forwarded-For");
        if (clientIp == null || clientIp.isEmpty() || "unknown".equalsIgnoreCase(clientIp)) {
            clientIp = request.getRemoteAddress() != null ? request.getRemoteAddress().getAddress().getHostAddress() : null;
        } else {
            clientIp = clientIp.split(",")[0].trim();
        }

        // 将客户端 IP 添加到请求头中
        ServerHttpRequest modifiedRequest = request.mutate()
                .header("X-Client-IP", clientIp)
                .build();

        return chain.filter(exchange.mutate().request(modifiedRequest).build());
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
}
```

---

### 注意事项

1. **安全性**：
   - 不要完全信任 `X-Forwarded-For` 头信息，因为它可以被客户端伪造。
   - 确保你的代理服务器或负载均衡器正确设置了 `X-Forwarded-For`。

2. **IPv6 支持**：
   - 如果客户端使用 IPv6 地址，确保你的应用能够正确处理 IPv6 格式。

3. **多级代理**：
   - 如果请求经过多级代理，`X-Forwarded-For` 可能包含多个 IP 地址，需要根据实际情况处理。

---

### 总结

- 如果请求不经过代理，直接使用 `request.getRemoteAddr()`。
- 如果请求经过代理，优先从 `X-Forwarded-For` 头信息中获取客户端 IP。
- 在 Spring Boot 中，可以通过 `HttpServletRequest` 或 `RequestContextHolder` 获取客户端 IP。

希望这些方法能帮助你成功获取客户端的 IP 地址！如果有其他问题，欢迎随时提问。