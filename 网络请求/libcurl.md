---
typora-copy-images-to: ../assets
---

#### 基本介绍

libcurl 是一个开源的客户端 URL 传输库，用 C 语言编写，用来在各种协议下发送和接收数据。

> https://curl.se/libcurl/
>
> https://curl.se/

核心特点：

- 协议支持极其广泛：支持 HTTP、HTTPS、FTP、FTPS、SCP、SFTP、SMTP、POP3、IMAP、LDAP、MQTT 等几乎所有常见的网络协议。
- 跨平台：可以在 Windows、Linux、macOS、iOS、Android 等几乎所有操作系统上编译运行。
- 线程安全：在正确使用的前提下，它是线程安全的。
- 高度可定制：支持 Cookie 处理、代理服务器（SOCKS/HTTP）、SSL 证书验证、断点续传、限制下载速度、表单提交等。
- 广泛的绑定：虽然是用 C 语言编写的，但几乎所有主流语言（Python、PHP、Java、Go、Rust 等）都有对应的 libcurl 绑定或封装。

#### 常规用法

libcurl 主要提供两套 API，适用于不同的场景：

+ Easy Interface（简易接口）：同步、阻塞。
+ Multi Interface（多路接口）：异步、非阻塞。

下面使用 libcurl 简易接口编写一个简单的 HTTP 请求，通常遵循以下五个步骤：

+ 全局初始化：`curl_global_init()`（整个程序运行期间调一次）。
+ 创建句柄：`curl_easy_init()`（创建一个新的请求实体，也就是拿到指针）。
+ 设置参数：`curl_easy_setopt()`（最重要的步骤，告诉 libcurl 目标 URL、超时时间、回调函数等）。
+ 执行任务：`curl_easy_perform()`（开始网络传输，此过程会阻塞）。
+ 清理收尾：`curl_easy_cleanup()` 和 `curl_global_cleanup()`。

```c
#include <stdio.h>
#include <curl/curl.h>

int main(void) {
    CURL *curl;
    CURLcode res;

    // 1. 初始化
    curl = curl_easy_init();
    if(curl) {
        // 2. 设置 URL
        curl_easy_setopt(curl, CURLOPT_URL, "https://www.example.com");

        // 3. 执行请求
        res = curl_easy_perform(curl);

        // 4. 检查错误
        if(res != CURLE_OK)
            fprintf(stderr, "请求失败: %s\n", curl_easy_strerror(res));

        // 5. 释放资源
        curl_easy_cleanup(curl);
    }
    return 0;
}
```

















此库用于HMI所有应用的网络库封装。

syan.com.cn

先安

https://www.syan.com.cn/?page_id=2351

https://www.syan.com.cn/?page_id=2353







https://www.syan.com.cn/?page_id=2323







- **安全 HTTP 通信**
- **PKI 双向认证**
- **SM2 证书类型**
- **系统 PKI 证书管理模块 PKICM**
- **AppKey + 时间戳 + TOTP 动态认证码**
- **车端到后台安全网关通信**