#### 简介

首先，区分下和 JUnit 的关系，JUnit 负责"运行测试 + 断言结果"，Mockito 负责"隔离依赖 + 控制依赖行为"，两者是一个互补的关系。

Mockito 是 Java 最流行的 Mock 框架之一。什么是 Mock？在测试对象 A 时，构造一些假的对象（Mock 对象）来模拟 A 与外部依赖之间的交互。Mock 对象的行为由我们预先设定，完全可控。

为什么要 Mock？因为某些对象是不容易构造的或不容易获取的。比如 Context 必须运行在 Android 环境中，纯 JUnit 无法创建；SharedPreferences，依赖 Android 系统服务，单元测试中不可用；Retrofit 的网络响应，真实请求不可控且不稳定。所以一旦出现外界依赖就需要 Mockito 来隔离。

另外，本文涉及的部分概念，已在《JUnit 4》中“单元测试”进行了详细介绍。

```java
// 典型用法
public class UserServiceTest {
    @Test
    public void shouldReturnUserById() {
        // 1.创建Mock对象
        UserRepository mockRepo = mock(UserRepository.class);

        // 2.预设行为（Stub）
        when(mockRepo.findById(1L)).thenReturn(Optional.of(new User("Alice")));

        // 3.注入到被测对象
        UserService service = new UserService(mockRepo);

        // 4.执行被测方法
        User result = service.getUser(1L);

        // 5.状态验证（优先）
        assertEquals("Alice", result.getName());

        // 6.行为验证（仅在必要时）
        verify(mockRepo).findById(1L);
    }
}
```

核心 API：

+ `mock()`：创建一个 Mock 对象。
+ `when(...).thenReturn(...)`：为 Mock 对象预设返回值（Stub 行为）。
+ `verify()`：验证某个方法是否被调用（行为验证）

#### 注解方式

```java
// 启用Mockito注解处理器，这样测试运行前会扫描所有@Mock字段、@InjectMocks字段，完成对象的Mock及注入
// 并可
@RunWith(MockitoJUnitRunner.class)
public class UserServiceTest {

    // 由Mockito自动创建Mock对象，等价于mock(UserRepository.class)
    @Mock
    private UserRepository mockRepo;

    // 由Mockito自动创建并将上面的@Mock对象注入，等价于new UserService(mockRepo)
    @InjectMocks
    private UserService service;

    @Test
    public void shouldReturnUserById() {
        // 1.预设行为（Stub）
        when(mockRepo.findById(1L)).thenReturn(Optional.of(new User("Alice")));

        // 2.执行被测方法
        User result = service.getUser(1L);

        // 3.状态验证（优先）
        assertEquals("Alice", result.getName());

        // 4.行为验证（仅在必要时）
        verify(mockRepo).findById(1L);
    }
}
```

`@RunWith(MockitoJUnitRunner.class)`，MockitoJUnitRunner 会在测试运行前会扫描所有 @Mock 字段、@InjectMocks 字段，完成对象的 Mock 及注入。并且为保证每个 @Test 方法拿到全新的 Mock，测试之间互不干扰。

@InjectMocks 是根据类型匹配，然后通过构造器注入、Setter注入、甚至直接反射设置，帮你注入。但它只能注入 @Mock 标记的对象，比如像你要传 String、int 这种，还是需要自己 new。

#### 参数匹配

有些时候 Stub 和 Verify 时，你不一定关心参数的精确值：

```java
// 精确匹配：只有传1L才生效
when(mockRepo.findById(1L)).thenReturn(user);

// 但如果想表达"传什么id都返回user"呢？用匹配器
when(mockRepo.findById(anyLong())).thenReturn(user);
```

常用匹配器：

```java
// ===== 通配匹配 =====
any()                  // 任意值（包括null）
anyLong()              // 任意long
anyString()            // 任意String
anyList()              // 任意List

// ===== 空值匹配 =====
isNull()               // 参数为null
isNotNull()            // 参数不为null

// ===== 精确匹配 =====
eq("Alice")            // 等于"Alice"
eq(100)                // 等于100

// ===== 条件匹配 =====
argThat(id -> id > 0)         // 自定义条件
startsWith("admin")           // String以"admin"开头

// ===== 类型匹配 =====
isA(User.class)        // 参数是User类型
```

典型用法：

```java
// Stub：任意 id 都返回同一个 user
when(mockRepo.findById(anyLong())).thenReturn(Optional.of(user));

// Stub：id > 0 才返回 user
when(mockRepo.findById(argThat(id -> id > 0)))
    .thenReturn(Optional.of(user));

// Verify：确认传入了以"admin"开头的邮箱
verify(mockSender).send(startsWith("admin"), anyString());
```

另外，精确值和匹配器是不能混用的，要么全用匹配器，要么全不用。

```java
when(mockRepo.findById(1L, anyString())).thenReturn(user);
// when(mockRepo.findById(1L)).thenReturn(user);
// when(mockRepo.findById(eq(1L), anyString())).thenReturn(user);

verify(mockSender).send(startsWith("admin"), "欢迎注册");
// verify(mockSender).send(startsWith("admin"), eq("欢迎注册"));
// verify(mockSender).send(startsWith("admin"), anyString());
// verify(mockSender).send("admin@test.com", "欢迎注册");
```

#### ArgumentCaptor 捕获参数

```java
// ==================== 被测代码 ====================
public class OrderService {
    private OrderRepository orderRepo;

    public OrderService(OrderRepository orderRepo) {
        this.orderRepo = orderRepo;
    }

    // 下单方法：内部创建 Order 对象并保存
    public void placeOrder(String userId, String productName, int quantity, double price) {
        Order order = new Order();
        order.setUserId(userId);
        order.setProductName(productName);
        order.setQuantity(quantity);
        order.setTotalPrice(quantity * price);
        order.setCreateTime(new Date());

        orderRepo.save(order);  // Order是在方法内部new出来组装的；想要验证发给OrderRepository的订单到底对不对
    }
}
```

```java
@Test
public void shouldSaveOrderWithCorrectTotalPrice() {
    OrderRepository mockRepo = mock(OrderRepository.class);
    OrderService service = new OrderService(mockRepo);

    // 执行下单
    service.placeOrder("user001", "键盘", 3, 100.0);

    // 声明捕获器
    ArgumentCaptor<Order> captor = ArgumentCaptor.forClass(Order.class);

    // 验证save被调用了，同时把传入的参数"抓"出来
    verify(mockRepo).save(captor.capture());

    // 验证发给OrderRepository的订单到底对不对
    Order savedOrder = captor.getValue();
    assertEquals("user001", savedOrder.getUserId());
    assertEquals("键盘", savedOrder.getProductName());
    assertEquals(3, savedOrder.getQuantity());
    assertEquals(300.0, savedOrder.getTotalPrice(), 0.01);
    assertNotNull(savedOrder.getCreateTime());
}
```

#### Mock 异常

+ 有返回值的方法：`when().thenThrow()`
+ void 方法：`doThrow().when()`

```java
// 有返回值的方法
when(mockRepo.findById(1L))
    .thenThrow(new RuntimeException("数据库连接失败"));
mockRepo.findById(1L);  // throws RuntimeException


// void方法
doThrow(new RuntimeException("发送失败"))
    .when(mockSender).send(anyString());
mockSender.send("hello");  // throws RuntimeException
```

还可以 Mock 连续调用：

```java
when(mockRepo.findById(anyLong()))
    .thenReturn(user)                              // 第1次调用：返回正常值
    .thenThrow(new RuntimeException("数据库挂了"))  // 第2次调用：抛异常
    .thenReturn(user);                              // 第3次及之后：返回正常值

mockRepo.findById(1L);  // → user
mockRepo.findById(2L);  // → throws RuntimeException
mockRepo.findById(3L);  // → user
```

验证异常：

```java
// 法一：assertThrows（更灵活）
@Test
public void shouldThrow() {
    PaymentException ex = assertThrows(PaymentException.class, () -> {
        service.pay(100.0);
    });
    assertEquals("支付失败", ex.getMessage());  // 还能断言异常详情
}

// 法二：@Test(expected = ...)
@Test(expected = PaymentException.class)
public void shouldThrow() {
    service.pay(100.0);
}
```

#### 静态方法测试

需要 Mock 的静态方法，大概率不该是静态方法，它往往依赖了外部状态（时间、数据库、网络等），应重构为可注入的依赖。但面对遗留代码或第三方库时，仍需了解如何 Mock。

```java
@Test
public void testGetWelcomeMessage() {
    // 必须用try-with-resources，否则静态Mock会泄漏到其他测试
    try (MockedStatic<DateUtils> mocked = mockStatic(DateUtils.class)) {
        mocked.when(DateUtils::today).thenReturn("2026-10-07");

        UserService service = new UserService();
        String result = service.getWelcomeMessage(new User("Alice"));

        assertEquals("Welcome Alice, today is 2026-10-07", result);
    }
    // try结束自动close，DateUtils.today()恢复原始行为
}
```

#### PowerMock

PowerMock 是一个 Mockito 的增强库，用于 Mock 早期 Mockito 做不到的事情，像静态方法、final 类/方法、构造函数、private 方法。因为早期 Mockito 通过 CGLIB（运行时动态生成类的子类）生成子类覆盖方法，static/final 无法被子类覆盖，所以做不到；PowerMock 直接修改字节码，不受继承限制。但现在 Mockito 3.5+ 已经支持（除了 private 方法；private 是内部实现细节，应通过 public 方法间接覆盖，太复杂则说明类该拆分），PowerMock 本身也被弃用。

另外想强调下，我们应该避免存在外部不可见的隐式依赖，应重构为构造器注入，让依赖关系显式可见。依赖关系应该是**显式的、可插拔的**，控制权在外部组装者手里（控制反转 IoC）。如果是**隐式的、写死的**，调用方就彻底失去了控制权。**如果一个类需要被 Mock，优先考虑提取接口**，但不是所有类都需要接口——只为"需要被替换"的依赖提取接口，过度接口化是一种浪费。

