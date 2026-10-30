---
typora-copy-images-to: ../assets
---

#### 基本介绍

OkHttp 是一个针对 Java 和 Android 平台的高效 HTTP 客户端，有这么几个核心特性：

- 连接池复用：如果 HTTP/2 不可用，OkHttp 会维护一个连接池，复用 TCP 连接，减少握手延迟。
- 支持 HTTP/2：允许同一主机的请求共享同一个 Socket，实现多路复用。
- 透明 GZIP 压缩：自动处理请求和响应的压缩，减少网络传输的数据量。
- 响应缓存：能够完全避免重复的网络请求，直接从本地读取缓存。
- 静默重连：当网络出现异常（如服务器有多个 IP）时，它会自动尝试备用地址，对用户透明。
- 拦截器机制：极其强大的扩展性，允许开发者监控、重写、重试任何请求。

核心组件：

- OkHttpClient：配置中心。负责超时设置、代理、拦截器注册等，建议在应用中作为**单例**存在。
- Request & Response：采用**建造者模式 (Builder)** 构建的不可变对象，分别代表 HTTP 的请求和响应。
- Call (RealCall)：请求的执行者。它将 Request 转换为 Response，支持同步 (execute) 和异步 (enqueue)。
- Dispatcher (分发器)：调度中心。它决定什么时候执行请求，维护了三个队列（就绪异步请求、运行中异步请求、运行中同步请求），并管理线程池。



| 对象         | 职责                                                |
| :----------- | :-------------------------------------------------- |
| Request      | 描述请求：URL、方法、Header、Body（纯数据，不可变） |
| Call         | 控制请求：什么时候发、同步还是异步、要不要取消      |
| OkHttpClient | 工厂：负责创建 Call，提供连接池、拦截器等配置       |



拦截器链：

当你发起一个请求时，这个请求并不会立刻飞到服务器，而是像工厂流水线一样，穿过一层层的“关卡”（拦截器），被加工处理后才发送出去；服务器返回响应时，也会倒序穿过这些关卡，最后交到你手里。

+ RetryAndFollowUpInterceptor（重试与重定向）：负责在网络波动时重试，或者根据 3xx 状态码进行重定向。
+ BridgeInterceptor（桥接）：负责补全请求头（如 Content-Type, Cookie, Keep-Alive, GZIP 等），并将返回的响应解压。
+ CacheInterceptor（缓存）：根据 HTTP 缓存协议（Cache-Control 等）判断是从磁盘读缓存还是发网络请求。
+ ConnectInterceptor（连接）：核心环节。负责打开与服务器的连接（从连接池找或者新建 TCP/TLS 握手）。
+ CallServerInterceptor（读写）：最后一个拦截器。负责通过 Socket 流向服务器写入请求数据，并读取响应。

以上是内置的 5 大拦截器，不能重写、替换，用户可以通过下面两个拦截器扩展逻辑。

+ Application Interceptor：注册在拦截器链的最外层（RetryAndFollowUpInterceptor 之前），无论底层是否发生重定向或重试，整个请求过程只执行一次。
+ Network Interceptor：注册在 ConnectInterceptor 和 CallServerInterceptor 之间，能看到网络上的原始流量（包括重试和重定向产生的多次请求）

#### 常规使用

1）依赖引入

```groovy
dependencies {
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
}
```

2）网络权限声明

```xml
<uses-permission android:name="android.permission.INTERNET"/>
```

3）OkHttpClient 

它是整个库的核心。注意，在应用中它应该是单例的，因为每个 Client 都有自己的连接池和线程池，重复创建会浪费资源。

```java
// 创建全局唯一的OkHttpClient实例，通常放在单例，或者Application中
OkHttpClient client = new OkHttpClient.Builder()
    .connectTimeout(10, TimeUnit.SECONDS) // 连接超时
    .readTimeout(10, TimeUnit.SECONDS)    // 读取超时
    .writeTimeout(10, TimeUnit.SECONDS)   // 写入超时
    .build();
```

4）GET 请求

同步请求 (Synchronous)，在 Android 中不能在主线程执行同步请求，否则会报 NetworkOnMainThreadException。

```java
// 创建一个Request对象，默认就是GET请求
Request request = new Request.Builder()
    .url("https://api.github.com/users/octocat")
    .header("User-Agent", "OkHttp Headers.java") // 添加 Header
    .build();

// 通过client的newCall方法创建Call对象（可以理解为待执行的请求任务）
// 使用try-with-resources确保response自动关闭（需要自己关闭，因为Java不知道你接下来还要不要用，所以只能让你自己决定什么时候关）
try (Response response = client.newCall(request).execute()) {
    // 判断HTTP状态码是否在2xx（200~299）范围内
    if (!response.isSuccessful()) {
        throw new IOException("Unexpected code " + response);
    }

    // 获取ResponseBody对象，然后获取响应字符串
    // .string()方法只能调用一次，会取出里面所有的内容
    String result = response.body().string();
    System.out.println(result);
} catch (IOException e) {
    // 不仅仅是client.newCall(request).execute()这里的错误，整个try里的都会catch，包括主动throw那里
    // 捕获网络异常（连接超时、DNS解析失败、非预期状态码等），打印堆栈
    e.printStackTrace();
}
```

异步请求 (Asynchronous)

这是 Android 开发中最常用的方式。

```java
client.newCall(request).enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {
        // 请求失败（如断网、超时），OkHTTP会自己close掉response
        e.printStackTrace();
    }

    @Override
    public void onResponse(Call call, Response response) throws IOException {
        // 请求成功（此回调仍在子线程中，更新UI需要切回主线程）
        try (ResponseBody responseBody = response.body()) {
            if (!response.isSuccessful()) {
                throw new IOException("Unexpected code " + response);
            }
            
            String result = responseBody.string();
            // 在这里处理结果...
        }
    }
});
```

5）POST 请求

POST 请求通常需要通过 RequestBody 携带数据，下面是提交普通的 JSON 数据。

```java
// 定义媒体类型，也就是告诉服务器解析的方式
MediaType JSON = MediaType.get("application/json; charset=utf-8");

// 构造JSON字符串（通常配合Gson使用）
String json = "{\"name\":\"tom\",\"age\":20}";
RequestBody body = RequestBody.create(json, JSON);

Request request = new Request.Builder()
    .url("https://example.com/api/user")
    .post(body)
    .build();

// 执行请求同上...
```

通过 Multipart 提交文件。

```java
File file = new File(getExternalFilesDir(null), "avatar.jpg");
MediaType mediaType = MediaType.parse("image/jpeg");

// 创建MultipartBody，把文字和文件打包在一起
RequestBody requestBody = new MultipartBody.Builder()
        .setType(MultipartBody.FORM) // 设置为表单类型
        .addFormDataPart("userId", "9527") // 普通文字参数
        .addFormDataPart("comment", "这是我的新头像") // 普通文字参数
        .addFormDataPart("file", file.getName(), RequestBody.create(file, mediaType)) // 文件参数
        .build();

Request request = new Request.Builder()
        .url("https://example.com/api/upload")
        .post(requestBody)
        .build();
client.newCall(request).enqueue(new Callback() { /* ... 同上 ... */ });
```

6）拦截器

有两种方式。

```java
OkHttpClient client = new OkHttpClient.Builder()
    // 注册应用拦截器，无论网络层发生了多少次重定向或重试，该拦截器从头到尾只会被调用一次
    // 典型用处：
    // 统一注入请求头，如Token、公共参数
    // 请求短路，命中缓存或Mock时直接返回，不往下走
    // 全局日志，打印原始请求和最终响应，观察完整交互过程
    // 状态码统一处理，如401自动刷新Token并重试
    .addInterceptor(new Interceptor() {
        @Override
        public Response intercept(Chain chain) throws IOException {
            Request request = chain.request();
            System.out.println("发出请求: " + request.url());

            long start = System.currentTimeMillis();
            Response response = chain.proceed(request);  // 发请求，等响应
            long end = System.currentTimeMillis();

            System.out.println("收到响应: " + response.code() + " 耗时 " + (end - start) + "ms");
            return response;
        }
    })
    // 注册网络拦截器，如果发生了重定向或网络重试，这个拦截器会针对每一次实际的网络请求执行一次
    // 典型用处：
    // 网络调试，能拿到真实的Connection对象（服务器 IP、TLS 版本、加密套件等）
    // 监控网络流量，统计实际消耗了多少字节的数据
    // 网络层日志，打印经过重定向/重试后的实际请求和响应，而非用户发出的原始请求
    .addNetworkInterceptor(new Interceptor() {
        @Override
        public Response intercept(Chain chain) throws IOException {
            Log.d("NetworkInterceptor", "实际网络传输");
            return chain.proceed(chain.request());
        }
    })
    .build();
```

































#### 底层原理

https://juejin.cn/post/7204480721435918391

https://github.com/enbandari/Android-Interview-2/blob/master/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/okhttp%E5%86%85%E6%A0%B8%E5%89%96%E6%9E%90.md

https://juejin.cn/post/6844904087788453896

https://juejin.cn/post/6898145227765186567

https://juejin.cn/post/7200591001903693882

https://www.mianshiya.com/tag/Okhttp

https://github.com/enbandari/Android-Interview-2/blob/master/%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/okhttp%E5%86%85%E6%A0%B8%E5%89%96%E6%9E%90.md



HttpURLConnection

Volley



----

### 第一阶段：基础入门（会用）

*目标：掌握 OkHttp 的基本配置和最常见的增删改查操作。*

1. **OkHttp 简介**什么是 OkHttp？它的优势是什么（连接池、GZIP、响应缓存）？与 HttpURLConnection 和 Volley 的对比。
2. **环境配置**添加 Gradle 依赖。网络权限申请（Android 权限声明）。
3. **核心 API 使用****GET 请求**：同步（execute）与异步（enqueue）的区别。**POST 请求**：提交表单数据（FormBody）。提交 JSON 数据（RequestBody.create）。文件上传（MultipartBody）。
4. **Response 处理**获取状态码、响应头。解析 Body（字符串、字节流、JSON 转换）。**注意点**：response.body().string() 的一次性使用限制。

### 第二阶段：进阶核心（深入 API）

*目标：掌握如何根据业务需求定制化网络客户端。*

1. **OkHttpClient 配置**单例模式的最佳实践。超时设置（Connect, Read, Write Timeout）。错误重连机制（retryOnConnectionFailure）。
2. **拦截器（Interceptors）—— OkHttp 的灵魂****应用拦截器（Application Interceptors）**与 **网络拦截器（Network Interceptors）**的区别。实战案例：添加通用 Header（如 Token、User-Agent）。打印网络日志（HttpLoggingInterceptor）。统一处理异常状态码。
3. **缓存机制**设置 Cache 目录和大小。强制读取缓存/强制读取网络（CacheControl）。HTTP 缓存协议在 OkHttp 中的体现（ETag, Last-Modified）。
4. **Cookie 管理**CookieJar 接口的使用。实现持久化 Cookie（自动登录逻辑）。

### 第三阶段：高级特性（性能与安全）

*目标：处理复杂的网络环境和高并发场景。*

1. **HTTPS 与安全**证书校验（自签名证书处理）。证书锁定（Certificate Pinning）。忽略 SSL 证书（仅用于测试环境）。
2. **并发调度（Dispatcher）**最大并发请求数、单主机最大请求数限制。线程池在 OkHttp 中的工作原理。
3. **WebSocket**如何建立长连接。消息的发送与监听。
4. **DNS 解析与协议控制**自定义 DNS（解决 DNS 劫持）。协议配置（HTTP/1.1, HTTP/2, QUIC/HTTP/3）。

### 第四阶段：原理剖析（源码级理解）

*目标：理解 OkHttp 的内部工作流，面试加分项。*

1. **整体流程图**从 call.execute() 到得到 Response 的完整生命周期。
2. **五大核心拦截器详解**RetryAndFollowUpInterceptor：重试与重定向。BridgeInterceptor：补全 Header，桥接应用与网络。CacheInterceptor：缓存策略执行。ConnectInterceptor：建立连接（寻找健康连接）。CallServerInterceptor：真正的数据读写。
3. **责任链模式（Chain of Responsibility）**源码分析：拦截器是如何层层传递并返回的。
4. **连接池原理（ConnectionPool）**如何复用连接（Keep-Alive）。清理无用连接的策略。

### 第五阶段：生态与实战（工程化）

*目标：将 OkHttp 应用到大型项目中。*

1. **与 Retrofit 配合**理解 Retrofit 是如何封装 OkHttp 的。如何在 Retrofit 中定制 OkHttpClient。
2. **辅助工具**使用 Chucker 在手机上直接查看网络请求日志。使用 Stetho 或 Charles 进行抓包调试。
3. **封装实战**封装一个通用的 OkHttpUtils 库（支持回调转换、自动解析 JSON、Loading 展示）。





