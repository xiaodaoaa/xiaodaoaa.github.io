下面是一个使用 C++ 和 OpenSSL 实现的简单的 TLS 加密客户端和服务端示例。

### 服务端代码 (C++)
服务端使用 OpenSSL 来创建 TLS 连接，并监听客户端的连接请求。

```cpp
#include <iostream>
#include <openssl/ssl.h>
#include <openssl/err.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

#define PORT 12345
#define CERT_FILE "server.crt"
#define KEY_FILE "server.key"

void init_openssl() {
    SSL_load_error_strings();
    OpenSSL_add_ssl_algorithms();
}

void cleanup_openssl() {
    EVP_cleanup();
}

SSL_CTX* create_context() {
    const SSL_METHOD* method = SSLv23_server_method();
    SSL_CTX* ctx = SSL_CTX_new(method);

    if (!ctx) {
        std::cerr << "Unable to create SSL context" << std::endl;
        ERR_print_errors_fp(stderr);
        exit(EXIT_FAILURE);
    }

    return ctx;
}

void configure_context(SSL_CTX* ctx) {
    if (SSL_CTX_use_certificate_file(ctx, CERT_FILE, SSL_FILETYPE_PEM) <= 0) {
        ERR_print_errors_fp(stderr);
        exit(EXIT_FAILURE);
    }

    if (SSL_CTX_use_PrivateKey_file(ctx, KEY_FILE, SSL_FILETYPE_PEM) <= 0) {
        ERR_print_errors_fp(stderr);
        exit(EXIT_FAILURE);
    }
}

int main() {
    init_openssl();
    SSL_CTX* ctx = create_context();
    configure_context(ctx);

    int sock;
    struct sockaddr_in addr;

    sock = socket(AF_INET, SOCK_STREAM, 0);
    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    addr.sin_addr.s_addr = INADDR_ANY;

    if (bind(sock, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
        perror("Unable to bind");
        exit(EXIT_FAILURE);
    }

    if (listen(sock, 1) < 0) {
        perror("Unable to listen");
        exit(EXIT_FAILURE);
    }

    while (1) {
        struct sockaddr_in client_addr;
        uint len = sizeof(client_addr);
        int client = accept(sock, (struct sockaddr*)&client_addr, &len);
        
        if (client < 0) {
            perror("Unable to accept");
            exit(EXIT_FAILURE);
        }

        SSL* ssl = SSL_new(ctx);
        SSL_set_fd(ssl, client);

        if (SSL_accept(ssl) <= 0) {
            ERR_print_errors_fp(stderr);
        } else {
            char buffer[1024] = {0};
            SSL_read(ssl, buffer, sizeof(buffer));
            std::cout << "Received: " << buffer << std::endl;
            SSL_write(ssl, "Hello, TLS Client!", 18);
        }

        SSL_shutdown(ssl);
        SSL_free(ssl);
        close(client);
    }

    close(sock);
    SSL_CTX_free(ctx);
    cleanup_openssl();
}
```

### 客户端代码 (C++)
客户端连接到服务器并通过 TLS 发送和接收数据。

```cpp
#include <iostream>
#include <openssl/ssl.h>
#include <openssl/err.h>
#include <arpa/inet.h>
#include <unistd.h>

#define PORT 12345
#define SERVER_IP "127.0.0.1"

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

int main() {
    init_openssl();
    SSL_CTX* ctx = create_context();

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

### 说明
1. **OpenSSL 库**：此示例使用 OpenSSL 库来处理 TLS 加密连接。
2. **证书和私钥**：需要生成 TLS/SSL 证书（`server.crt`）和私钥（`server.key`）。可以使用 OpenSSL 来生成这些文件。以下是生成自签名证书的命令：
    ```bash
    openssl req -x509 -newkey rsa:2048 -keyout server.key -out server.crt -days 365 -nodes
    ```
3. **端口**：服务器监听的端口号为 12345，客户端需要连接到这个端口。
4. **双向通信**：客户端向服务器发送消息，服务器会返回一条简单的响应。

### 运行步骤
1. 生成证书和私钥文件。
2. 编译和运行服务器和客户端代码：
    - 编译：`g++ -o server server.cpp -lssl -lcrypto && g++ -o client client.cpp -lssl -lcrypto`
    - 运行服务端：`./server`
    - 运行客户端：`./client`