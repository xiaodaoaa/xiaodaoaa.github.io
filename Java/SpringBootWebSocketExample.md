在使用 `javax.websocket.server.ServerEndpoint` 和 Spring Boot 时，通常会结合 `ServerEndpointExporter` 来自动注册 WebSocket 端点。下面是一个完整的示例，展示如何在 Spring Boot 项目中使用 `ServerEndpointExporter` 和 `ServerEndpoint` 注解来创建一个 WebSocket 服务器。

### 1. 创建 Spring Boot 项目

首先，创建一个新的 Spring Boot 项目，可以使用 Spring Initializr 并添加以下依赖：

- Spring Web
- Spring WebSocket

或者直接在 `pom.xml` 文件中添加以下依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
<dependency>
    <groupId>javax.websocket</groupId>
    <artifactId>javax.websocket-api</artifactId>
    <version>1.1</version>
</dependency>
```

### 2. 创建 WebSocket 服务器端点

创建一个 WebSocket 服务器端点，处理客户端的连接和消息：

```java
import javax.websocket.OnClose;
import javax.websocket.OnMessage;
import javax.websocket.OnOpen;
import javax.websocket.Session;
import javax.websocket.server.ServerEndpoint;
import java.io.IOException;
import java.util.Collections;
import java.util.HashSet;
import java.util.Set;

@ServerEndpoint("/chat")
public class NotifyWebSocket {
    private static final List<Session> SESSION_LIST = new CopyOnWriteArrayList<>();

    @OnOpen
    public void onOpen(Session session) throws Exception {
        SESSION_LIST.add(session);
    }

    @OnMessage
    public void onMessage(String msg) {
        log.debug("收到信息 {}", msg);
    }

    public void sendInfo(String msg) {
        SESSION_LIST.forEach(session -> {
            try {
                session.getBasicRemote().sendText(msg);
            } catch (IOException e) {
                log.error("信息:{}发送客户端{}失败", msg, session.getId());
            }
        });
    }


    @OnError
    public void onError(Session session, Throwable error) {
        log.error("系统错误[{}]", session.getUserPrincipal().getName(), error);
        try {
            session.getBasicRemote().sendText(error.getMessage());
        } catch (IOException e) {
            log.error("错误信息发送失败", e);
        }
    }

    /**
     * 连接关闭调用的方法
     */
    @OnClose
    public void onClose(Session session) {
        SESSION_LIST.remove(session);
        log.debug("a websocket closed!session id is:{}", session.getId());
    }
}
```

### 3. 配置 `ServerEndpointExporter`

在 Spring Boot 应用中，需要一个 `ServerEndpointExporter` bean 来自动注册带有 `@ServerEndpoint` 注解的 WebSocket 端点：

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.server.standard.ServerEndpointExporter;

@Configuration
public class WebSocketConfig {
    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }
}
```

### 4. 启动类

确保你的主类标注了 `@SpringBootApplication` 注解：

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class WebSocketDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(WebSocketDemoApplication.class, args);
    }
}
```

### 5.发送消息

```java
@Service
public class TestServiceImpl implements TestService {
    @Autowired
    private NotifyWebSocket notifyWebSocket;

    @Override
    public String exec() {
        notifyWebSocket.sendInfo("spring websocket test.");
        return "send success";
    }
}
```

### 6. 创建前端客户端

创建一个简单的 HTML 文件作为前端客户端来测试 WebSocket 连接。新建一个文件 `src/main/resources/static/index.html`：

```html
<!DOCTYPE html>
<html>
<head>
    <title>WebSocket Chat</title>
    <script type="text/javascript">
        var ws;
        function connect() {
            ws = new WebSocket("ws://localhost:8080/chat");

            ws.onmessage = function(event) {
                var chatBox = document.getElementById("chatBox");
                chatBox.value += event.data + "\n";
            };

            ws.onopen = function() {
                console.log("Connected");
            };

            ws.onclose = function() {
                console.log("Disconnected");
            };
        }

        function sendMessage() {
            var message = document.getElementById("message").value;
            ws.send(message);
            document.getElementById("message").value = '';
        }
    </script>
</head>
<body onload="connect()">
    <h1>WebSocket Chat</h1>
    <textarea id="chatBox" cols="100" rows="20" readonly></textarea><br>
    <input type="text" id="message" size="100"/>
    <button onclick="sendMessage()">Send</button>
</body>
</html>
```

### 7. 运行应用程序

启动 Spring Boot 应用程序，然后打开浏览器访问 `http://localhost:8080/index.html`。你应该能够看到一个简单的聊天界面，多个浏览器窗口之间可以实时发送和接收消息。

### 总结

这个示例展示了如何在 Spring Boot 项目中使用 `javax.websocket.server.ServerEndpoint` 注解来定义 WebSocket 端点，并结合 `ServerEndpointExporter` 自动注册这些端点。通过这种方式，可以实现实时的双向通信，非常适合聊天应用、在线游戏和实时数据推送等场景。