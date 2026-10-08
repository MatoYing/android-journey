### 单元测试

#### 什么是单元测试

单元测试（Unit Testing）是针对代码中最小可测试单元（通常是一个函数、一个方法、一个类）编写的自动化测试，验证它在各种输入下的行为是否符合预期。

```java
// 被测代码
public class Calculator {
    int add(int a, int b) {
        return a + b;
    }
}

// 单元测试
@Test
public void test_add() {
    Calculator calc = new Calculator();
    assertThat(calc.add(2, 3)).isEqualTo(5);
    assertThat(calc.add(-1, 1)).isEqualTo(0);
    assertThat(calc.add(0, 0)).isEqualTo(0);
}
```

单元测试的核心原则是：只测试一个单元，隔离其他依赖。为了隔离外部依赖，确保这些依赖不影响验证逻辑，我们会用到以下手段来做 Test Double。

> https://martinfowler.com/articles/mocksArentStubs.html

这篇文章其实提到了五点：

| 替身（Test Double） | 职责                                                         |
| :------------------ | :----------------------------------------------------------- |
| **Dummy**           | 凑参数的，传进去但从不用，比如传个 `null`                    |
| **Stub**            | 预设返回值，"有人问你就答这个"，`when().thenReturn()`        |
| **Mock**            | 验证行为，"你应该被调用一次"，`verify()`                     |
| **Spy**             | 能干活还偷偷记录，事后可以查"被调了几次、参数是什么"         |
| **Fake**            | 手写一个轻量实现，比如用 HashMap 代替数据库，用真实逻辑去模拟 |

以一个例子来解释下：

```java
// 依赖接口
public interface OrderRepository {
    Order findById(int id);
    void save(Order order);
}

public interface EmailService {
    void send(String email, String subject, String body);
}

// 被测类
public class OrderService {
    private final OrderRepository repo;
    private final EmailService emailService;

    public OrderService(OrderRepository repo, EmailService emailService) {
        this.repo = repo;
        this.emailService = emailService;
    }

    // Query方法：返回值，不改变状态
    public double getOrderTotal(int orderId) {
        Order order = repo.findById(orderId);
        return order.getItems().stream()
            .mapToDouble(Item::getPrice)
            .sum();
    }

    // Command方法：改变状态，无返回值
    public void cancelOrder(int orderId) {
        Order order = repo.findById(orderId);
        order.setStatus("CANCELLED");
        repo.save(order);
        emailService.send(order.getUserEmail(), "订单取消", "您的订单已取消");
    }
}
```

##### 1）Dummy

测 `getOrderTotal()` 只需 `OrderRepository`，不需要通知用户，因此 `emailService` 传 `null` 充当 Dummy，也就是一个占位符，代码不调用它，测试也不管它。

```java
@Test
public void testGetOrderTotal() {
    OrderRepository repoStub = mock(OrderRepository.class);
    when(repoStub.findById(1)).thenReturn(someOrder);

    // emailService传null，getOrderTotal根本不用它
    OrderService service = new OrderService(repoStub, null);  // ← null 就是 Dummy

    double total = service.getOrderTotal(1);
    assertThat(total).isEqualTo(300.0);
}
```

##### 2）Fake

使用纯内存集合代替真实的 MySQL 读写，拥有可工作的轻量级完整逻辑。

```java
public class FakeOrderRepository implements OrderRepository {
    private Map<Integer, Order> store = new HashMap<>();

    public Order findById(int id) { return store.get(id); }
    public void save(Order order) { store.put(order.getId(), order); }
}

public class FakeEmailService implements EmailService {
    public List<String> sentEmails = new ArrayList<>();

    public void send(String email, String subject, String body) {
        sentEmails.add(email + ": " + subject);
    }
}
```

```java
@Test
public void testCancelOrder_withFakes() {
    FakeOrderRepository fakeRepo = new FakeOrderRepository();
    FakeEmailService fakeEmail = new FakeEmailService();

    Order order = new Order(1, "user@test.com");
    fakeRepo.save(order);

    OrderService service = new OrderService(fakeRepo, fakeEmail);
    service.cancelOrder(1);

    // 从Fake里取出来看，状态真的改了
    assertThat(fakeRepo.findById(1).getStatus()).isEqualTo("CANCELLED");
    // 邮件真的"发"了
    assertThat(fakeEmail.sentEmails).contains("user@test.com: 订单取消");
}
```

##### 3）Stub

测 `getOrderTotal()`（Query 方法），只需 `findById()` 返回固定数据。

```java
@Test
public void testGetOrderTotal() {
    OrderRepository repoStub = mock(OrderRepository.class);
    when(repoStub.findById(1)).thenReturn(someOrderWithItems);  // 预设值

    OrderService service = new OrderService(repoStub, null);
    double total = service.getOrderTotal(1);

    assertThat(total).isEqualTo(300.0);
}
```

##### 4）Spy

Spy 核心是“暗中记账”，测试结束后用普通断言读取记账本（状态验证）。

```java
// 专门记录外发邮件历史
public class SpyEmailService implements EmailService {
    public final List<String> sentMessages = new ArrayList<>();

    @Override
    public void send(String email, String subject, String body) {
        sentMessages.add(email + ":" + subject);
    }
}

@Test
public void testCancelOrder_withSpy() {
    SpyEmailService emailSpy = new SpyEmailService();
    OrderRepository repoStub = mock(OrderRepository.class);
    when(repoStub.findById(1)).thenReturn(someOrder);

    OrderService service = new OrderService(repoStub, emailSpy);
    service.cancelOrder(1);

    // 查账本（依然是状态断言）：检查记下来的数据对不对
    assertThat(emailSpy.sentMessages).hasSize(1);
    assertThat(emailSpy.sentMessages.get(0)).isEqualTo("user@test.com:订单取消");
}
```

##### 5）Mock

测 `cancelOrder()`（Command 方法），验证 `save()` 和 `send()` 有没有被正确调用，验证调用行为。

```java
@Test
public void testCancelOrder() {
    OrderRepository repoMock = mock(OrderRepository.class);
    EmailService emailMock = mock(EmailService.class);

    when(repoMock.findById(1)).thenReturn(someOrder);

    OrderService service = new OrderService(repoMock, emailMock);
    service.cancelOrder(1);

    // 验证行为
    verify(repoMock).save(argThat(o -> "CANCELLED".equals(o.getStatus())));
    verify(emailMock).send("user@test.com", "订单取消", "您的订单已取消");
    verifyNoMoreInteractions(repoMock, emailMock);
}
```

##### 总结

首先，Dummy 只是一个不重要的占位参数，完全可以忽略；而 Spy 本质上是带记录功能的 Stub（或者说是一种简易的 Mock），在现代测试中也可以归纳到 Stub/Mock 的范畴里。因此，我们主要面对的是 Fake、Stub、Mock 的选择：

+ 自己的内部逻辑用**“状态验证”**，外部边界用**“行为验证”**。对于自己写的逻辑（包括 void 方法），优先通过 Stub（`assert()`）来验证，关注的是**"结果对不对"**，不需要关注里面有没有调用什么方法。只有面对无法观察**内部状态**的外部系统时，才退而求其次用 Mock（`verify()`）验证**"有没有调用"**。
+ Fake 强调的是手写一个接口的轻量实现，比如用内存数据结构（如 HashMap）代替真实的外部依赖（如数据库）。Stub 能做的其实 Fake 都能做，选哪个取决于调用的复杂度，当该类仅仅被调用 1-2 次，用 Stub 更方便，一两行 `when/thenReturn` 搞定；当改类被多次调用，有存有取、有状态流转，用 Fake，写一次实现类，所有测试复用，不用每个测试都配一堆 `when`。

现代 Mock 框架（如 Mockito）把这些概念统一到了一个 API 里。 核心方法就是 `mock()`，同一个对象既能当 Stub 用，也能当 Mock 用，取决于你怎么用它。而Fake 需要自己手写一个接口的实现。

#### 单元测试的意义

+ 减少 Bug：验证最小可测试单元（函数、方法、类）在各种输入条件下的行为是否符合预期，在最早的阶段拦截缺陷。
+ 支撑重构信心：重构在软件工程中的经典定义是"在不改变外部行为（Public Signature & Behavior）的前提下改善内部结构"（当然不是说 public 签名不能改，该改还是得改）。有完善的单元测试覆盖，改完后跑一遍就能快速发现逻辑错误，降低引入回归缺陷（新改动导致旧功能坏掉）的风险。
+ 驱动更好的设计：为了让代码可测试，你会自然地倾向于面向对象、高度解耦、职责清晰的写法，依赖必须抽象成接口以便测试时注入 Mock，代码的协作关系在设计之初就被规划得很清楚。测试困难的代码，往往就是设计有问题的代码。

#### TDD

Test-Driven Development，测试驱动开发。核心思想：先写测试，再写实现。本质是“设计工具”，而非单纯的“测试工具”。步骤是三步循环（红-绿-重构）：

+ Red → 写一个会失败的测试（因为功能还没实现）
+ Green → 写最少的代码让测试通过（哪怕写得很丑）
+ Refactor → 测试全绿后，在不改变行为的前提下改善代码结构

实际开发中，大部分时间是在 Red → Green 之间快速循环，每轮都通过新测试暴露不足、再补实现让它通过。Refactor 不是每轮都做，而是攒到合适时机（代码出现重复、结构不清晰、功能告一段落）再统一整理。下面是个例子：

1）Red，测试失败 = 红。这是对的，说明你定义了一个还没实现的需求。

```java
@Test
void 营业时间内应返回true() {
    BusinessHoursChecker checker = new BusinessHoursChecker();
    assertTrue(checker.isOpen(10));  // 编译都过不了，方法还不存在
}
```

2）Green，测试通过 = 绿。代码丑没关系，先让它对。

```java
public class BusinessHoursChecker {
    public boolean isOpen(int hour) {
        return true;  // 最粗暴的实现，但测试能过
    }
}
```

3）继续 Red，补充新测试暴露不足，测试又红了，逼着你完善实现：

```java
@Test
void 凌晨3点应返回false() {
    BusinessHoursChecker checker = new BusinessHoursChecker();
    assertFalse(checker.isOpen(3));  // 失败了！因为上面永远返回 true
}
```

```java
public boolean isOpen(int hour) {
    return hour >= 9 && hour <= 22;
}
```

4）Refactor，两个测试都绿了，代码也够简洁，本轮完成。接下来继续补充更多边界测试（9 点整、22 点整、负数……），不断重复 Red → Green → Refactor 循环。

**TDD 的好处：**

- 需求想得更清楚：写测试就是在定义"这段代码应该做什么"，逼你在动手前先把需求想透。
- 帮助设计接口：从使用者视角写测试，接口的合理性在写实现之前就被验证了，避免空想设计出糟糕的 API。
- 减少 Bug、支撑重构：每行代码都是被测试"催"出来的，小步前进、持续验证，缺陷在最早阶段被拦截；测试覆盖率天然高，每次重构都有测试兜底，不担心回归。

**TDD 的争议：**

- 门槛高。
- 投入开发资源（时间和精力）通常会更多。
- 可能限制整体设计：测试用例在代码设计之前编写，如果过于关注局部行为，可能忽略更优的整体架构。

**TDD 与 AI：**

结合目前大火的 AI 智能体工作流框架 Superpowers，它内部就是使用 TDD 进行编程，AI 最大的问题不是"不会写代码"，而是写得太快、太多，且无法自我验证正确性。TDD 恰好解决了这个核心矛盾：

- 约束生成冲动：强制先写测试再写实现，小步前进，防止 AI 一次堆出大量无法验证的代码。
- 自动验证正确性：测试是可执行的客观标准，AI 能自己判断"写对没有"，不需要人类逐行 review；AI 也无法在测试面前"幻觉"或"讨好"，过了就是过了，没过就是没过。
- 持续可重构：AI 可以在测试保护下大胆优化代码结构，改坏了立刻亮红灯，具备了自主安全重构的能力。

### JUnit 4

JUint 是 Java 编程语言的单元测试框架，用于编写和运行可重复的自动化测试。

#### 基本步骤

当新声明一个类后，按如下操作，之后点击弹窗的“Create New Test”就会创建出来。

![image-20261006172439533](../../../../../Library/Application Support/typora-user-images/image-20261006172439533.png)

+ @Test：标识测试方法，让 JUnit 可以识别到。加上后，左面会出现“绿色三角”运行按钮。
+ @Ignore：跳过某个测试方法。

```java
import org.junit.Test;
import static org.junit.Assert.*;

public class CalculatorTest {
    @Test
    // @Ignore("This test is not yet implemented")
    public void testAdd() {
        assertEquals(4, new Calculator().add(2, 2));
    }
}
```

点击运行按钮会弹出右面的选项，这里 Coverage 代表了覆盖率，统计的是被测试执行到的代码行数占总代码行数的比例。

![image-20261006173805807](../../../../../Library/Application Support/typora-user-images/image-20261006173805807.png)

这里 Branch 是指代码里的"岔路口"，`if/else`、`switch`、`while` 这类有判断条件的地方。

![image-20261006174515704](../../../../../Library/Application Support/typora-user-images/image-20261006174515704.png)

运行完 Coverage 会出现下面这样的颜色条

+ 绿色：该行代码在测试运行期间已被执行。如果是分支语句，说明所有可能的分支走向（true 和 false），都在测试用例中被覆盖到了。   
+ 黄色：该行代码被执行了，但分支没有完全覆盖。例如图中的 `else if (b == 0)`，测试用例可能只触发了其中一种情况（比如只测试了条件为 true 或只测试了 false），没有把所有判断分支都测全。   
+ 红色：该行代码在测试过程中完全没有被执行过。

![image-20261006202153222](../../../../../Library/Application Support/typora-user-images/image-20261006202153222.png)

另外这里 Branch 有 8 条：

| 来源               | 分支数 |
| :----------------- | :----- |
| `if (a == 0)`      | 2      |
| `else if (b == 0)` | 2      |
| `a != 0`           | 2      |
| `b != 0`           | 2      |
| **总计**           | **8**  |

#### 常用断言

| 断言                      | 用途     | 失败条件  |
| :------------------------ | :------- | :-------- |
| `assertEquals(a, b)`      | 值相等   | a ≠ b     |
| `assertNotEquals(a, b)`   | 值不等   | a = b     |
| `assertTrue(x)`           | 为真     | x = false |
| `assertFalse(x)`          | 为假     | x = true  |
| `assertNull(x)`           | 为空     | x ≠ null  |
| `assertNotNull(x)`        | 非空     | x = null  |
| `assertSame(a, b)`        | 同一引用 | 不同对象  |
| `assertArrayEquals(a, b)` | 数组相等 | 内容不同  |
| `@Test(expected=...)`     | 抛异常   | 没抛      |
| `@Test(timeout=...)`      | 不超时   | 超时了    |
| `assertThat + Matcher`    | 灵活匹配 | 不匹配    |

使用 Demo：

```java
// ── 值相等 ──────────────────────────────────────
Assert.assertEquals(5, result);                    // 整数相等
Assert.assertEquals("加法不对", 5, result);         // 带提示信息，Terminal里会显示
Assert.assertEquals(3.14, result, 0.01);            // 浮点数，第三个参数是误差范围

// ── 值不等 ──────────────────────────────────────
Assert.assertNotEquals(0, result);                  // 结果不应该等于0

// ── 布尔判断 ────────────────────────────────────
Assert.assertTrue(validator.isValid("hello"));      // 应该为true
Assert.assertFalse(validator.isValid(""));          // 应该为false

// ── 空值判断 ────────────────────────────────────
Assert.assertNull(repository.findById("999"));      // 找不到应该返回null
Assert.assertNotNull(repository.findById("admin")); // 存在，不该为null

// ── 引用判断（是不是同一个对象）────────────────────
Assert.assertSame(configA, configB);               // 单例，同一引用
Assert.assertNotSame(userA, userB);                // 不是同一个对象

// ── 数组相等 ────────────────────────────────────
Assert.assertArrayEquals(new int[]{1, 2, 3}, Sorter.sort(input));

// ── 断言抛异常 ──────────────────────────────────
@Test(expected = IllegalArgumentException.class)
public void testDivideByZero() {
    calculator.divide(1, 0);                 // 没抛这个异常就算失败
}

// ── 断言不超时 ──────────────────────────────────
@Test(timeout = 2000)
public void testQuickOp() {
    service.process(data);                   // 超过2秒就算失败
}

// ── assertThat + Matcher（灵活匹配）─────────────
Assert.assertThat(names, hasItem("Alice"));         // 包含某个元素
Assert.assertThat(names, hasSize(3));               // 集合长度为3
Assert.assertThat(names, contains("Alice", "Bob")); // 精确顺序匹配
Assert.assertThat(text, startsWith("Hello"));       // 字符串以"Hello"开头
Assert.assertThat(value, greaterThan(10));          // 大于10
Assert.assertThat(obj, instanceOf(User.class));     // 是某个类型的实例
```

#### 生命周期

```
@BeforeClass          ← 整个类开始前执行一次（静态方法）
│
├── @Before           ← 每个 @Test 方法之前
│   └── @Test         ← 执行测试
│       └── @After    ← 每个 @Test 方法之后
│
└── @Before
│   └── @Test
│       └── @After
│
└── ...
@AfterClass            ← 整个类结束后执行一次（静态方法）
```

Demo：

```java
public class UserServiceTest {

    // ── 类级别：只执行一次 ──────────────────────
    @BeforeClass
    public static void setUpOnce() {
        System.out.println("@BeforeClass — 数据库连接（只建一次）");
    }

    @AfterClass
    public static void tearDownOnce() {
        System.out.println("@AfterClass — 关闭数据库连接");
    }

    // ── 方法级别：每个@Test都执行一轮 ─────────
    @Before
    public void setUp() {
        System.out.println("@Before — 准备数据");
    }

    @After
    public void tearDown() {
        System.out.println("@After — 清理数据");
    }

    // ── 测试方法 ─────────────────────────────────
    @Test
    public void testFindUser() {
        System.out.println("@Test testFindUser");
    }
    
    @Test
    public void testRegister() {
        System.out.println("@Test testRegister");
    }
}
```

#### 参数化测试

参数化测试就是同一个测试方法，用不同的数据跑多遍，不用复制粘贴写一堆相似的测试。

步骤：

1. 类上加 `@RunWith(Parameterized.class)`（`@RunWith` 就是选一个运行器，不同运行器决定测试"怎么跑"。不加就用默认的，需要特殊能力（参数化、Spring 注入）时才加对应的）。
2. 写一个静态方法，用 `@Parameters` 标注，返回测试数据。
3. 提供构造函数，会通过它把每组数据传进去（构造函数参数顺序要和 `@Parameters` 一致）。
4. 写一个 `@Test` 方法，用传入的数据来断言。

```java
import org.junit.Test;
import org.junit.runner.RunWith;
import org.junit.runners.Parameterized;
import org.junit.runners.Parameterized.Parameters;

import java.util.Arrays;
import java.util.Collection;

import static org.junit.Assert.assertEquals;

@RunWith(Parameterized.class)
public class CalculatorAddTest {

    // 传入的参数
    private int a;
    private int b;
    private int expected;

    // 构造函数接收每组数据
    public CalculatorAddTest(int a, int b, int expected) {
        this.a = a;
        this.b = b;
        this.expected = expected;
    }

    // 提供测试数据
    @Parameters(name = "{0} + {1} = {2}")  // Terminal里会根据这个显示出来，比如`testAdd[1 + 1 = 2]     Pass`
    public static Collection<Object[]> data() {  // 需要是static，因为它在创建实例前就被调用了
        return Arrays.asList(new Object[][] {
            // {a, b, expected}
            { 1,  1,  2},
            { 0,  0,  0},
            {-1,  1,  0},
            {-3, -5, -8},
            {100, 200, 300}
        });
    }

    // 一个测试方法，将会跑5遍（因为有5组数据）
    @Test
    public void testAdd() {
        Calculator calc = new Calculator();
        assertEquals(expected, calc.add(a, b));
    }
}
```

注意：一个类里只要用了 `@RunWith(Parameterized.class)`，所有 `@Test` 都会按参数化跑，想写普通测试尽量分出去单独一个类，否则会跟着 Parameters 的数量重复跑多遍。

#### JUnit 4 和 5 的区别

+ 参数化测试更简洁，不需要上文讲到的步骤，不用换 Runner、不用拆类，`@ParameterizedTest` 一个注解搞定。
+ 可以指定多个 Runner。
+ 支持 `@Nested` 分组、条件测试（按条件决定跑不跑）、Lambda 断言消息。
+ 模块化架构，可以选择支持 JUnit 4。
