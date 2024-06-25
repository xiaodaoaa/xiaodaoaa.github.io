Hutool 是一个非常实用的 Java 工具库，其中的 `HttpUtil` 类提供了丰富的 HTTP 请求和响应处理功能，使得在 Java 应用中进行 HTTP 请求变得更加便捷和灵活。下面是一个详细的示例，展示了如何使用 `HttpUtil` 发送 HTTP 请求并处理响应。

### 引入 Hutool 库

首先，需要将 Hutool 库引入你的项目中。可以通过 Maven 或者 Gradle 进行依赖管理。

#### Maven
```xml
<dependency>
    <groupId>cn.hutool</groupId>
    <artifactId>hutool-http</artifactId>
    <version>5.7.2</version> <!-- 注意替换为最新版本 -->
</dependency>
```

#### Gradle
```groovy
implementation 'cn.hutool:hutool-http:5.7.2' // 注意替换为最新版本
```

### 使用示例

下面是一个基本的示例，展示了如何使用 `HttpUtil` 发送 HTTP GET 请求，并处理响应的内容。

```java
import cn.hutool.http.HttpRequest;
import cn.hutool.http.HttpResponse;
import cn.hutool.http.HttpUtil;

public class HttpUtilExample {
    public static void main(String[] args) {
        // 发送 GET 请求并获取响应
        String url = "https://jsonplaceholder.typicode.com/posts/1";
        HttpResponse response = HttpUtil.createGet(url).execute();

        // 打印响应状态码和响应内容
        System.out.println("Response status: " + response.getStatus());
        System.out.println("Response body:\n" + response.body());

        // 可选：获取响应头信息
        System.out.println("Response headers:\n" + response.headers());
    }
}
```

### 示例解释

1. **创建 HTTP 请求**：
   - 使用 `HttpUtil.createGet(url)` 创建一个 GET 请求对象，其中 `url` 是请求的目标地址。

2. **执行 HTTP 请求**：
   - 使用 `.execute()` 方法执行 HTTP 请求，并返回一个 `HttpResponse` 对象，其中包含了请求的响应信息。

3. **处理响应**：
   - `HttpResponse` 对象可以获取响应的状态码（`getStatus()`）、响应体内容（`body()`）和响应头信息（`headers()`）等。

4. **示例输出**：
   - 该示例会输出请求的响应状态码、响应体内容和响应头信息，方便开发人员进行进一步处理和分析。

### 其他常用方法

除了 GET 请求之外，`HttpUtil` 还支持 POST、PUT、DELETE 等各种 HTTP 请求方法，以及文件上传、表单提交等功能。以下是几个常用方法的示例：

#### 发送 POST 请求

```java
import cn.hutool.http.HttpRequest;
import cn.hutool.http.HttpResponse;
import cn.hutool.http.HttpUtil;
import cn.hutool.http.Method;

public class HttpUtilPostExample {
    public static void main(String[] args) {
        String url = "https://jsonplaceholder.typicode.com/posts";
        String postData = "{\"title\": \"foo\", \"body\": \"bar\", \"userId\": 1}";

        // 发送 POST 请求并获取响应
        HttpResponse response = HttpUtil.createRequest(Method.POST, url)
                                        .body(postData)
                                        .execute();

        // 处理响应
        System.out.println("Response status: " + response.getStatus());
        System.out.println("Response body:\n" + response.body());
    }
}
```
#### HttpPostClient示例
```java
    private String httpPostClient(String url, String token, String body) {
        log.info("url::{}, token::{}, reqBody::{}", url, token, body);
        String response = null;
        try {
            Map<String, String> headers = new HashMap<>();
            headers.put("Content-Type", "application/json");
            headers.put("access_token", token);
            HttpResponse httpResponse = HttpUtil.createPost(url).body(body).addHeaders(headers).execute();
            log.info("httpResponse:{}", JSONObject.toJSONString(httpResponse));
            String resCode = Optional.ofNullable(httpResponse)
                    .map(m -> JSONUtil.parseObj(m.body()))
                    .map(m -> m.getStr("code"))
                    .orElseThrow(() -> new RuntimeException("request failed."));
            if ("200".equals(resCode)) {
                response = httpResponse.body();
                log.info("{}", response);
            } else {
                log.info("请求失败::{}", resCode);
                log.info("{}", httpResponse.body());
            }
        } catch (Exception e) {
            log.info("请求失败.", e);
        }
        return response;
    }
```

#### 发送带参数的 GET 请求

```java
import cn.hutool.http.HttpRequest;
import cn.hutool.http.HttpResponse;
import cn.hutool.http.HttpUtil;

public class HttpUtilGetParamsExample {
    public static void main(String[] args) {
        String url = "https://jsonplaceholder.typicode.com/posts";
        // 设置 GET 请求的参数
        HttpRequest request = HttpUtil.createGet(url)
                                      .form("userId", "1")
                                      .form("title", "foo");

        // 执行请求并获取响应
        HttpResponse response = request.execute();

        // 处理响应
        System.out.println("Response status: " + response.getStatus());
        System.out.println("Response body:\n" + response.body());
    }
}
```

#### 发送带form参数的POST请求

```java
    private ThirdpartyTokenDto getToken() {
        ThirdpartyTokenDto response = null;
    
        String url = "http://127.0.0.1/queryToken";
        Map<String, String> headers = new HashMap<>();
        headers.put("Content-Type", "application/x-www-form-urlencoded");
    
        Map<String, Object> form = new HashMap<>();
        form.put("clientIdd", "123456");
        form.put("clientSecret", "123456789abcd");
        try {
            HttpResponse httpResponse = HttpUtil.createPost(url).form(form).addHeaders(headers).execute();
            response = JSONObject.parseObject(httpResponse.body(), ArtemisTokenDto.class);
        } catch (Exception e) {
    
        }
        return response;
    }
```


### 总结

使用 Hutool 的 `HttpUtil` 类，可以简化 Java 应用中的 HTTP 请求处理，提供了丰富的方法和选项，支持常见的请求和响应操作，使得与 Web 服务的交互更加方便和高效。以上示例展示了基本的使用方式，你可以根据具体的需求进一步扩展和定制功能。