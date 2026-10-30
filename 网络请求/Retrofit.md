---
typora-copy-images-to: ../assets
---

#### 基本介绍

Retrofit 是一个类型安全的 RESTful HTTP 网络请求框架，基于动态代理将 Java/Kotlin 接口注解（如 @GET、@POST、@Header 等）转化为实际的 HTTP 请求。其底层网络通信本质上由 OkHttp 完成，Retrofit 负责的是：

+ 接口封装：通过注解声明 Url、请求方法、参数、Header 等信息；
+ 请求适配：借助 CallAdapter 将请求结果转换为不同返回类型（如 `Call<T>`、`Observable<T>`、`Flowable<T>`）；
+ 数据转换：借助 Converter（如 GsonConverterFactory）将响应体自动反序列化为 Java/Kotlin 对象。

整体流程：App 调用 Retrofit 接口 $\to$ Retrofit 根据注解构建请求 $\to$ OkHttp 执行网络请求 $\to$ 服务端返回响应 $\to$ OkHttp 将原始响应交还 Retrofit $\to$ Converter 解析响应体 $\to$ CallAdapter 适配返回类型 $\to$ 最终结果交付给调用方。

#### 常规用法

1）添加依赖

```groovy
dependencies {
    // Retrofit 核心库
    implementation 'com.squareup.retrofit2:retrofit:2.11.0' 
    // Gson 转换器（自动把 JSON 字符串转成 Java/Kotlin 对象）
    implementation 'com.squareup.retrofit2:converter-gson:2.11.0'
    // OkHttp 拦截器（用于打印网络日志，可选但强烈推荐）
    implementation 'com.squareup.okhttp3:logging-interceptor:4.12.0'
}
```

2）网络权限

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

3）定义实体类

```java
// {
  // "id": 1024,
  // "name": "Alex",
  // "email": "alex@example.com"
// }
public class User {
    private int id;
    private String name;
    private String email;

    // Getters and Setters ...
}
```

4）定义 API 接口

```java
public interface ApiService {
    // GET请求
    // @Query会自动拼接成：users?limit=10
    @GET("users")
    Call<List<User>> getUsers(@Query("limit") int limit);

    // GET动态路径
    // @Path会自动替换URL中的{id}
    // 上买呢和这种，用啥取决于后端怎么定义，不用纠结这个
    @GET("users/{id}")
    Call<User> getUserById(@Path("id") int userId);

    // POST请求
    // @Body会自动将User对象转成JSON字符串放在请求体中
    @POST("users/create")
    Call<ResponseBody> createUser(@Body User user);
}
```

5）创建 Retrofit 实例

使用单例模式来管理 Retrofit 实例。

```java
public class RetrofitClient {
    private static final String BASE_URL = "https://api.example.com/";
    private static Retrofit retrofit = null;

    public static ApiService getApiService() {
        if (retrofit == null) {
            // 创建OkHttpClient并添加日志拦截器
            // 用来在控制台打印网络请求和响应的完整信息，方便调试
            HttpLoggingInterceptor logging = new HttpLoggingInterceptor();
            logging.setLevel(HttpLoggingInterceptor.Level.BODY); // BODY是最详细的，可以打印完整请求与响应体
            
            OkHttpClient client = new OkHttpClient.Builder()
                    .addInterceptor(logging)
                    .build();

            // 创建Retrofit实例
            retrofit = new Retrofit.Builder()
                    .baseUrl(BASE_URL)
                    .client(client) // 使用我们自定义的OkHttp
                    .addConverterFactory(GsonConverterFactory.create()) // 配置数据转换器，服务器返回给你的JSON字符串，通过Gson自动解析成你定义好的实体类
                    .build();
        }
        return retrofit.create(ApiService.class);
    }
}
```

6）发起网络请求

异步请求：

```java
ApiService apiService = RetrofitClient.getApiService();

// 初始化Call
Call<User> call = apiService.getUserById(1024);

// 放入队列进行异步请求
call.enqueue(new Callback<User>() {
    @Override
    public void onResponse(Call<User> call, Response<User> response) {
        if (response.isSuccessful() && response.body() != null) {
            User user = response.body();
            // 此时已回到主线程，可以直接更新UI
            textView.setText(user.getName());
        } else {
            // 请求成功，但服务器返回了错误码
            Log.e("Retrofit", "错误码: " + response.code());
        }
    }

    @Override
    public void onFailure(Call<User> call, Throwable t) {
        // 网络异常（如断网、超时）
        Log.e("Retrofit", "网络失败: " + t.getMessage());
    }
});
```

同步请求（会阻塞线程，必须在子线程中运行）：

```java
// 必须放在new Thread()或线程池中
try {
    Response<User> response = call.execute();
    User user = response.body();
} catch (IOException e) {
    e.printStackTrace();
}
```

7）接入 RxJava

```groovy
dependencies {
    // RxJava3核心库与Android扩展库
    implementation 'io.reactivex.rxjava3:rxjava:3.1.8'
    implementation 'io.reactivex.rxjava3:rxandroid:3.0.2'
    // Retrofit的RxJava3适配器
    implementation 'com.squareup.retrofit2:adapter-rxjava3:2.11.0'
}
```

默认情况下，Retrofit 只能返回 `Call<T>`。为了让它能返回 RxJava 的 `Observable<T>` 或 `Single<T>`，必须在这里注入工厂：

```java
Retrofit retrofit = new Retrofit.Builder()
        .baseUrl("https://api.example.com/")
        .addConverterFactory(GsonConverterFactory.create())
        .addCallAdapterFactory(RxJava3CallAdapterFactory.create()) // 添加RxJava3适配器
        .build();
```

修改 API 接口的返回值：

```java
public interface ApiService {
    // 以前：Call<User> getUserById(@Path("id") int userId);
    // 现在：改成返回RxJava的Single或者Observable
    @GET("users/{id}")
    Single<User> getUserById(@Path("id") int userId);
}
```

发起请求。接入 RxJava 后，Retrofit 的返回类型从 `Call<T>` 变为 `Observable<T>` 或 `Single<T>` 等响应式类型，从而可以利用 RxJava 提供的线程调度（subscribeOn / observeOn）和操作符链（如 map、flatMap、retry、zip 等）对请求流程和结果进行灵活编排。

```java
ApiService apiService = retrofit.create(ApiService.class);
apiService.getUserById(1024)
        .subscribeOn(Schedulers.io())               // 指定网络请求在IO线程（子线程）执行
        .observeOn(AndroidSchedulers.mainThread())  // 指定结果回调在Android主线程执行
        .subscribe(new SingleObserver<User>() {     // 订阅并处理结果
            @Override
            public void onSuccess(User user) {
                // 请求成功，直接在主线程更新UI
                textView.setText(user.getName());
            }

            @Override
            public void onError(Throwable e) {
                // 请求失败，统一捕获各类网络异常（如 404、超时、断网等）
                Log.e("RxJava", "出错啦: " + e.getMessage());
            }
        });
```

#### 网络库层次关系 

Retrofit

OkHTTP

Volley

HTTPURLConnection

https://www.jianshu.com/p/77d418e7b5d6

https://juejin.cn/post/7408795408863363082





























----

更详细的注解：https://juejin.cn/post/6844903588821467143



### 第一阶段：基础入门（快速上手）

**目标：掌握 Retrofit 的基本配置和最简单的请求发送。**

1. 
2. **Retrofit 简介**什么是 Retrofit？为什么要用它（对比 OkHttp、Volley）？Retrofit、OkHttp、Gson 之间的关系。
3. **环境配置**引入依赖（Retrofit, Converter-Gson）。网络权限申请。
4. **核心三部曲**定义接口 (API Interface)：使用 @GET 注解。创建实例 (Retrofit Instance)：配置 baseUrl 和 Converter。发起请求：同步执行 (execute) 与 异步执行 (enqueue)。
5. **数据转换**Converter 的作用：自动将 JSON 转为实体类（POJO/Data Class）。

### 第二阶段：注解全解（核心功法）

**目标：能够熟练定义各种复杂的 HTTP 请求。**

1. 
2. **请求方法注解**@GET, @POST, @PUT, @DELETE, @PATCH。
3. **标记类注解**@FormUrlEncoded (表单提交)。@Multipart (文件上传)。@Streaming (大文件下载，防止内存溢出)。
4. **参数类注解（重点）**URL 路径：@Path。查询参数：@Query, @QueryMap。请求体：@Body。表单字段：@Field, @FieldMap。文件上传：@Part, @PartMap。
5. **Header 操作**静态 Header：@Headers。动态 Header：@Header, @HeaderMap。

### 第三阶段：深度整合（进阶实战）

**目标：结合现代开发模式（Kotlin + 响应式）。**

1. 
2. **配合 Kotlin 协程 (Coroutines)**使用 suspend 定义接口函数。异常处理：try-catch 与 runCatching。
3. **适配器 (CallAdapters)**为什么需要 CallAdapter？整合 RxJava (RxJava3CallAdapterFactory)。整合 Kotlin Flow (自定义或第三方库)。
4. **数据转换器 (Converters) 详解**常用：Gson, Moshi, Kotlin Serialization。自定义 Converter：处理特殊的加密或格式需求。

### 第四阶段：OkHttp 联动（底层调优）

**目标：理解 Retrofit 的“引擎”，掌握拦截器和安全配置。**

1. 
2. **自定义 OkHttpClient**设置超时 (Timeout)。配置缓存 (Cache)。
3. **拦截器 (Interceptors) 的妙用**日志拦截器 (HttpLoggingInterceptor)。公共参数拦截器（添加 API Key、Token、设备信息）。动态修改 URL。
4. **安全性**HTTPS 证书校验（SSL/TLS）。Token 刷新机制 (Authenticator 接口)。

### 第五阶段：高级技巧与架构设计

**目标：在大型项目中优雅地封装 Retrofit。**

1. 
2. **网络请求层封装**单例模式管理 Retrofit 实例。Repository 模式：ViewModel -> Repository -> Retrofit。
3. **错误处理机制**统一封装 Result 状态类（Loading, Success, Error）。业务状态码（非 2xx）的统一拦截与解析。
4. **多 BaseUrl 处理**如何在一个项目中切换不同的服务器地址。
5. **性能优化**连接池管理。减少反射开销。

### 第六阶段：原理与源码（面试加分）

**目标：理解 Retrofit 的核心设计模式。**

1. 
2. **核心设计模式****动态代理 (Dynamic Proxy)**：深入理解 Retrofit 如何将接口转换为请求对象（面试必问）。**外观模式 (Facade)**：Retrofit 类的统一入口设计。**适配器模式 (Adapter)**：CallAdapter 的实现。
3. **工作流程分析**从 create() 到 enqueue() 的完整链路。ServiceMethod 的解析与缓存。
