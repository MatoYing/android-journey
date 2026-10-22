#### 基本介绍

Gson 是 Google 开发的一个 JSON 解析库，实现 Java 对象与 JSON 数据之间的相互转换。

| 维度         | Gson                          | Jackson                       | Fastjson                       |
| ------------ | ----------------------------- | ----------------------------- | ------------------------------ |
| **开发者**   | Google                        | FasterXML                     | 阿里巴巴                       |
| **解析速度** | 慢（反射）                    | 快（反射 + ASM 字节码生）     | 最快（反射 + ASM 字节码生）    |
| **包体积**   | 极小（约 200KB）              | 大（分模块，完整引入约 2MB+） | 中等（约 600KB）               |
| **易用性**   | 高（非常符合直觉）            | 低（配置复杂，学习曲线陡）    | 高（静态方法，一行代码）       |
| **安全性**   | 高                            | 高                            | 低（历史上多次曝出严重安全漏） |
| **适用场景** | 中小型项目、传统 Android 开发 | 性能要求极高、后端开发        | 国内追求速度的小项目           |

（ASM 更底层，直接碰字节码；反射更上层，通过 JVM 提供的元数据 API。Jackson 首次用反射探路分析结构，再用 ASM 生成高效代码并缓存；后续直接复用，避开反射开销）

总的来说 Android 选择了它，主要就是因为体积小，功能够用。

#### 常规用法

1）引入依赖

```groovy
dependencies {
    implementation 'com.google.code.gson:gson:2.10.1'
}
```

2）序列化

即对象 $\rightarrow$ JSON。

我们需要定义一个实体类：

```java
public class User {
    public String name;
    public int age;
    public String email;
    
    // 省略 getter/setter/toString
}
```

通过 `toJson()` 转换：

```java
User user = new User("张三", 25, "zhangsan@example.com");

Gson gson = new Gson();
// 使用toJson()实现
String jsonString = gson.toJson(user);

// 输出结果：{"name":"张三","age":25,"email":"zhangsan@example.com"}
Log.d("Gson", jsonString);
```

3）反序列化

即 JSON $\rightarrow$ 对象，通过 `fromJson()` 实现：

```java
String jsonString = "{\"name\":\"李四\",\"age\":22,\"email\":\"lisi@example.com\"}";

Gson gson = new Gson();
// 使用fromJson()实现
User user = gson.fromJson(jsonString, User.class);

// 此时 user 对象已经填充了对应的数据
Log.d("Gson", "姓名：" + user.getName());
```

3）BaseResponse

后端服务器返回的 JSON 数据通常有一套固定的外壳格式。为了不让 Android 端为每个接口都重复写 `code` 和 `msg` 变量，我们利用 Java 的泛型，把这个公共的外壳抽离出来，这就是 `BaseResponse<T>`。

```json
{
  "code": 200,          // 状态码：200表示成功，401表示未登录，500表示服务器崩溃
  "msg": "请求成功",     // 提示信息：用来给用户展示或者写日志
  "data": { ... }       // 真正的核心业务数据，每次请求返回的格式都不同
}
```

```java
public class BaseResponse<T> {
    private int code;
    private String msg;
    private T data; // T代表具体的业务数据类型，可以是任意对象

    // Getter和Setter方法
    public int getCode() { return code; }
    public String getMsg() { return msg; }
    public T getData() { return data; }
    
    // 快捷判断请求是否成功
    public boolean isSuccess() {
        return code == 200; // 假设200是后端约定的成功码
    }
}
```

编译器在编译时擦除了泛型信息，Gson 在运行时执行 fromJson 时已经拿不到 `<User>` 这样的类型参数，所以无法知道该反序列化成什么具体类型。

在运行时，`BaseResponse.class` 传给 Gson 时，Gson 只知道你要一个 BaseResponse 对象， 但它根本不知道里面的 `data` 到底是 `User`、还是是 `Order`。 最终 Gson 会把 `data` 默认解析成一个 `LinkedTreeMap`，当你后面尝试 `response.getData().getName()` 时，程序就会直接崩溃。

Java 虽然会擦除方法和局部变量的泛型，但是对于“类定义”中的泛型，它的信息会被保留在字节码的元数据（Metadata）中。TypeToken 正是利用这一点，`new TypeToken<BaseResponse<User>>() {}` 创建了一个继承 TypeToken 匿名类。在这个新创建的子类中，它明确继承了父类的泛型 `BaseResponse<User>`，这时候，Java 编译器就会把 `BaseResponse<User>` 这个完整的类型信息，写进这个子类的字节码元数据里。最后，调用 `.getType()` 方法，`TypeToken` 内部就会通过反射去读取自己字节码里的元数据，将 `BaseResponse<User>` 类型信息重新提取出来，之后 Gson 就可以利用这个，直接 `fromJson()` 即可。（为什么 Gson 不直接反射拿 `BaseResponse<User>` 呢？因为运行时这个 `<User>` 已经彻底消失了，拿不到）（也可以拿到 `LinkedTreeMap` 一个一个 get，然后自己强转）

```java
String jsonString = "{\"code\":200,\"msg\":\"成功\",\"data\":{\"name\":\"张三\",\"age\":18}}";

// {}表明创建了一个TypeToken的匿名内部类（也就是它的子类），
Type type = new TypeToken<BaseResponse<User>>(){}.getType();

BaseResponse<User> response = new Gson().fromJson(jsonString, type);

if (response.isSuccess()) {
    User user = response.getData(); // 直接拿到User对象
    System.out.println(user.getName()); // 张三
} else {
    System.out.println("错误原因：" + response.getMsg());
}
```

```java
tring jsonString = "{\"code\":200,\"msg\":\"成功\",\"data\":[{\"title\":\"新闻1\"},{\"title\":\"新闻2\"}]}";

// 明确指出T的位置是List<News>集合
Type type = new TypeToken<BaseResponse<List<News>>>(){}.getType();

BaseResponse<List<News>> response = new Gson().fromJson(jsonString, type);

if (response.isSuccess()) {
    List<News> newsList = response.getData(); // 直接拿到新闻列表
    System.out.println(newsList.size()); // 2
}
```

只有序列化的时候需要 TypeToken 去转，序列化直接 `toJson()` 就可以了。 

4）自定义 JSON 键名

实际开发中，服务器返回的字段名往往不符合 Android 的命名规范，使用 `@SerializedName` 可以实现映射：

```java
public class User {
    @SerializedName("user_id")
    private int id; // 解析时，“user_id”会自动赋值给id变量

    @SerializedName({"user_name", "username"}) // 也可以支持多个备用键名
    private String name;
}
```

5）过滤不想序列化的字段

如果有些本地变量不需要转成 JSON，或者不需要从 JSON 中解析，可以直接加上 Java 的 `transient` 关键字。

```java
public class User {
    private String name;
    private transient String localToken; // 这个字段不会被转成JSON，也不会从JSON中解析出来
}
```

Gson 还提供了 `@Expose`，这个能做更多精细化的控制，而不是一刀切。

```java
// 必须这样创建Gson对象，@Expose才会生效
Gson gson = new GsonBuilder()
    .excludeFieldsWithoutExposeAnnotation()
    .create();

// 默认，双向都参与
@Expose(serialize = true, deserialize = true)
// 转JSON时没有，但反序列化包含
@Expose(serialize = false)
// 解析JSON时忽略，但序列化包含
@Expose(deserialize = false)
```

6）个性化配置

默认的 `new Gson()` 是最基础的配置。如果需要处理日期格式、保留 null 值等，可以使用 `GsonBuilder`：

```java
Gson gson = new GsonBuilder()
        .serializeNulls() // 默认情况下null值的字段不显示，配置后会显示出来如"email":null
        .setDateFormat("yyyy-MM-dd HH:mm:ss") // 统一配置日期的格式化
        .setPrettyPrinting() // 格式化输出（让JSON字符串带缩进，方便调试看日志）
        .create();
```

7）混淆问题

混淆为了防止 App 被人反编译，也为了减小 APK 的体积，Android 的混淆工具（ProGuard 或 R8）会把你的代码进行“重命名”和“精简“。

```java
// 混淆前（你写的代码）
public class User {
    private String name;
    private String email;
}

// 混淆后（编译器生成的代码）
public class a {
    private String a;
    private String b;
}
```

Gson 是非常依赖于变量名的，因为要和 JSON 匹配。需要在 `proguard-rules.pro` 中进行配置：

```shell
# 保护Gson库自带的注解和内部类不被混淆
-keep attributes *Annotation*
-keep class com.google.gson.** { *; }

# 保护你自己的Bean类不被混淆；也可以为每个变量名上加@SerializedName
-keep class com.example.app.bean.** { *; }
```

#### 泛型擦除

泛型信息只存在于编译阶段，编译器会检查泛型类型的安全性，然后将所有的泛型参数替换为它们的原始类型（如果泛型参数没有设置上限（如 `<T>`），则会被擦除为 Object；如果设置了上限（如 `<T extends Number>`），则会被擦除为 Number）。在编译完成后，所有的泛型信息都会被“擦除”，在生成的字节码（.class 文件）中是不包含泛型参数的。

```java
// 编译前
public class Box<T> {
    private T value;
    public void set(T value) { this.value = value; }
    public T get() { return value; }
}

Box<String> box = new Box<>();
box.set("Hello");
String s = box.get();
```

```java
// 编译后实际等价于
public class Box {
    private Object value; // T被替换为Object
    public void set(Object value) { this.value = value; }
    public Object get() { return value; }
}

Box box = new Box();
box.set("Hello");
String s = (String) box.get(); // 编译器自动插入了强制类型转换
```

为什么要这么做？Java 在 JDK 5 才引入泛型，为了能让在更早版本下编译的旧代码，在 JDK 5 的虚拟机上依然能跑，所以引入了这个概念。

好处是向后兼容了，但也引出一些坏处：

==TODO 解决方式==

+ 无法使用 instanceof：你不能写 `if (obj instanceof List<String>)`。编译器会报错，因为运行时只剩下了 List，无法判断具体的泛型类型。

+ 无法获取 Class 对象：`List<String>.class` 是不存在的。无论泛型是什么，`getClass()` 返回的永远是原始类型 `ArrayList.class`。

+ 不能 `new T()`、`new T[10]`：编译器不知道“目标 T”是否有无参构造函数，也不知道到底要分配多少内存。

+ 堆污染：即一个泛型引用指向了一个并不符合其类型约束的对象。

  ```java
  List<String> strings = new ArrayList<>();
  List rawList = strings;  // rawList是一个裸类型，编译器不知道它里面该放什么类型，所以检查不出来
  rawList.add(123);       // 编译不报错，但往String列表里塞了个Integer
  
  String s = strings.get(0); // 运行时抛出ClassCastException
  ```

+ 方法重载冲突：泛型擦除会导致这两个方法的签名在编译后变得一模一样。

  ```java
  public void print(List<String> list) {}
  public void print(List<Integer> list) {} // 报错：两个方法在擦除后签名相同
  ```

+ 强制的基本类型包装：因为泛型擦除后变成 Object，而像 int、double、long 它们的基本类型不是 Object，没法实现泛型擦除，所以、你不能写 `List<int>`，只能写 `List<Integer>`。这会导致频繁的装箱与拆箱，影响性能。





