在典型的 TLS 连接中，客户端并不总是需要证书和密钥，具体取决于服务器的配置和安全需求。通常有以下几种模式：

1. **单向认证 (Server Authentication)**:
   - **使用场景**: 这是最常见的情况，客户端只验证服务器的证书，而服务器不验证客户端的身份。
   - **流程**: 客户端向服务器请求连接，服务器发送自己的证书给客户端，客户端验证服务器的证书是否合法（例如，证书是否受信任的 CA 签发、证书是否过期等）。验证成功后，双方建立加密连接。
   - **证书**: 客户端不需要自己的证书，只需服务器的公钥证书来验证服务器。

2. **双向认证 (Mutual Authentication / Client and Server Authentication)**:
   - **使用场景**: 这种模式用于高安全性环境，例如金融、企业内部应用等。在这种模式下，服务器不仅向客户端提供证书，客户端也需要向服务器提供自己的证书，双方相互验证对方的身份。
   - **流程**: 客户端与服务器相互发送证书并验证对方的证书。服务器需要验证客户端的证书是否有效、是否由受信任的 CA 签发等。
   - **证书**: 客户端和服务器都需要拥有自己的证书和私钥。

### 简单 TLS 客户端不需要证书的实现

在单向认证的典型场景下，客户端不需要证书和私钥。上面提供的客户端代码就是这种情况，客户端只需要连接到服务器，并且依赖服务器的证书来建立安全连接。

### 配置双向认证（客户端需要证书和私钥）

如果需要双向认证，客户端将需要提供自己的证书和私钥。以下是如何在客户端配置证书的示例。

#### 客户端代码 (双向认证)
```cpp
#include <iostream>
#include <openssl/ssl.h>
#include <openssl/err.h>
#include <arpa/inet.h>
#include <unistd.h>

#define PORT 12345
#define SERVER_IP "127.0.0.1"
#define CLIENT_CERT_FILE "client.crt"
#define CLIENT_KEY_FILE "client.key"

void init_openssl() {
    SSL_load_error_strings();
    OpenSSL_add_ssl_algorithms();
}

void cleanup_openssl() {
    EVP_cleanup();
}

SSL_CTX* create_context() {
    const SSL_METHOD* method = SSLv23_client_method();
    SSL_CTX* ctx = SSL_CTX_new(method);

    if (!ctx) {
        std::cerr << "Unable to create SSL context" << std::endl;
        ERR_print_errors_fp(stderr);
        exit(EXIT_FAILURE);
    }

    return ctx;
}

void configure_context(SSL_CTX* ctx) {
    // 客户端加载自己的证书和私钥
    if (SSL_CTX_use_certificate_file(ctx, CLIENT_CERT_FILE, SSL_FILETYPE_PEM) <= 0) {
        ERR_print_errors_fp(stderr);
        exit(EXIT_FAILURE);
    }

    if (SSL_CTX_use_PrivateKey_file(ctx, CLIENT_KEY_FILE, SSL_FILETYPE_PEM) <= 0) {
        ERR_print_errors_fp(stderr);
        exit(EXIT_FAILURE);
    }

    // 验证客户端证书与私钥是否匹配
    if (!SSL_CTX_check_private_key(ctx)) {
        std::cerr << "Private key does not match the public certificate" << std::endl;
        exit(EXIT_FAILURE);
    }
}

int main() {
    init_openssl();
    SSL_CTX* ctx = create_context();
    configure_context(ctx);  // 配置客户端证书和密钥

    SSL* ssl;
    int server;
    struct sockaddr_in addr;

    server = socket(AF_INET, SOCK_STREAM, 0);
    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    inet_pton(AF_INET, SERVER_IP, &addr.sin_addr);

    if (connect(server, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
        perror("Unable to connect");
        exit(EXIT_FAILURE);
    }

    ssl = SSL_new(ctx);
    SSL_set_fd(ssl, server);

    if (SSL_connect(ssl) <= 0) {
        ERR_print_errors_fp(stderr);
    } else {
        SSL_write(ssl, "Hello, TLS Server!", 18);
        char buffer[1024] = {0};
        SSL_read(ssl, buffer, sizeof(buffer));
        std::cout << "Received: " << buffer << std::endl;
    }

    SSL_shutdown(ssl);
    SSL_free(ssl);
    close(server);
    SSL_CTX_free(ctx);
    cleanup_openssl();
}
```

### 生成客户端证书和密钥
为了进行双向认证，客户端也需要生成证书和私钥。与服务器证书生成类似，以下是生成客户端证书的命令：

```bash
openssl req -newkey rsa:2048 -nodes -keyout client.key -x509 -days 365 -out client.crt
```

然后将这些证书文件（`client.crt` 和 `client.key`）配置到客户端的代码中。

### 双向认证的优势
双向认证增加了安全性，因为它确保了不仅客户端可以验证服务器的身份，服务器也可以验证客户端的身份。这可以防止未授权的客户端连接到服务器，适用于敏感数据传输场景。

### 总结
- **单向认证**：客户端只验证服务器的证书，客户端不需要证书。
- **双向认证**：客户端和服务器互相验证对方的证书，客户端需要证书和私钥。