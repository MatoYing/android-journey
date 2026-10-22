#### 概述

Robolectric 是一个用于 Android 单元测试的开源框架，它的核心目标是解决普通单元测试在 JVM（非 Android 模拟器或真机）上无法访问 Android API 及资源文件。

简单 Demo：

```java
@RunWith(RobolectricTestRunner.class) // 指定Robolectric的测试Runner
@Config(sdk = 28)                     // 可指定模拟的Android SDK版本
public class MainActivityTest {

    @Test
    public void clickingButton_shouldChangeResultsViewText() {
        // 1.构建Activity
        MainActivity activity = Robolectric.buildActivity(MainActivity.class).create().get();

        // 2.获取View并模拟点击
        Button button = activity.findViewById(R.id.button);
        button.performClick();

        // 3.断言验证
        TextView results = activity.findViewById(R.id.results);
        assertThat(results.getText().toString()).isEqualTo("Clicked!");
    }
}
```

#### 原理

Robolectric 的关键设计是 Shadow 机制。它不会真正调用 Android 系统底层，而是用 Java 编写的"影子"类来模拟这些行为，它的本质是一种运行时替换技术。通过自定义 ClassLoader 在运行时拦截 Android 类加载，用纯 Java 的 Shadow 类替换真实 Android 框架实现，让 Android 代码在普通 JVM 上运行。Robolectric 为每个 Android 类（Activity、View、Looper 等）都准备了对应的 Shadow 实现，模拟其行为但不依赖真实 Android Runtime。

```
// JVM 正常加载类
Class<?> clazz = Class.forName("android.app.Activity");
// 尝试从android.jar加载 → 在纯JVM上会报错（找不到类）

// Robolectric 的类加载过程
┌─────────────────────────────────────┐
│         你的测试代码                   │
│   new Activity()                    │
└─────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────┐
│      Robolectric 自定义 ClassLoader   │
│                                     │
│   1. 检查是否有对应的 Shadow 类        │
│   2. 拦截对 Android 类的调用           │
│   3. 将调用转发到 Shadow 实现          │
└─────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────┐
│      实际执行的"类"                   │
│   ShadowActivity（纯 Java 实现）      │
└─────────────────────────────────────┘
```

```java
// Robolectric内部大致逻辑（伪代码）
class RobolectricClassLoader extends ClassLoader {
    
    @Override
    protected Class<?> findClass(String name) {
        if (isAndroidClass(name)) {
            // 1.加载Android类的stub版本
            Class<?> androidClass = loadStubClass(name);
            
            // 2.将Android类中的方法调用重定向到Shadow（用了Instrumentation的技术，与Android Instrumentation不同，在运行时修改字节码）
            byte[] modifiedBytecode = instrument(androidClass);
            
            // 3.定义修改后的类
            return defineClass(name, modifiedBytecode, 0, modifiedBytecode.length);
        }
        return super.findClass(name);
    }
}

// 原始Android类的方法
public class Activity {
    public void setContentView(int layoutResID) {
        // 原始实现...
    }
}

// Robolectric注入后的大致效果
public class Activity {
    public void setContentView(int layoutResID) {
        // 检查是否有Shadow，如果有则调用Shadow的方法
        if (Shadows.hasShadow(this)) {
            Shadows.getShadow(this).setContentView(layoutResID);
            return;
        }
        // 原始实现（在JVM上不会执行到）
    }
}
```

#### 基本使用

##### 依赖 

```java
android {
    testOptions {
        unitTests {
            // 在默认情况下，Android的单元测试（test/目录下，运行在JVM上的测试）是无法访问Android资源文件的
            // 将res/资源合并到单元测试中，单元测试就能正常读取res/下的资源了
            includeAndroidResources = true
        }
    }
}

dependencies {
    testImplementation 'org.robolectric:robolectric:4.11.1'
    testImplementation 'androidx.test:core:1.5.0'
    testImplementation 'androidx.test.ext:junit:1.1.5'
}
```

##### Activity 测试

```java
@RunWith(RobolectricTestRunner.class)  // 必须指定
@Config(sdk = 28)                      // 指定模拟的SDK版本
public class MainActivityTest {

    @Test
    public void activityNotNull() {
        // 构建并启动 Activity
        MainActivity activity = Robolectric.buildActivity(MainActivity.class).create().get();
        
        // 断言
        assertThat(activity).isNotNull();
    }
}
```

常用注解：

+ `@RunWith(RobolectricTestRunner.class)`：指定测试 Runner（必须）。
+ `@Config(sdk = 28)`：指定模拟的 Android SDK 版本。
+ `@Config(qualifiers = "zh-rCN")`：模拟特定语言/区域环境（横竖屏也可以）。

##### 生命周期测试

```java
@RunWith(RobolectricTestRunner.class)
public class ActivityLifecycleTest {
    @Test
    public void testActivityLifecycle() {
        ActivityController<MainActivity> controller = Robolectric.buildActivity(MainActivity.class);
        
        // 依次触发生命周期方法
        MainActivity activity = controller.get();
        
        controller.create();   // onCreate()
        controller.start();    // onStart()
        controller.resume();   // onResume()
        controller.pause();    // onPause()
        controller.stop();     // onStop()
        controller.destroy();  // onDestroy()
    }
  
    @Test
    public void testActivityWithIntent() {
        Intent intent = new Intent();
        intent.putExtra("key", "value");
        
        MainActivity activity = Robolectric.buildActivity(MainActivity.class, intent)
            .create()  // 模拟触发onCreate()
            .get();  // 拿到该Activity
        
        assertThat(activity.getIntent().getStringExtra("key")).isEqualTo("value");  // 验证Activity正确读取了Intent参数
    }
}
```

##### Fragment 测试

```java
@RunWith(RobolectricTestRunner.class)
public class MyFragmentTest {

    @Test
    public void testFragment() {
        MyFragment fragment = new MyFragment();
        
        // 启动Fragment（使用FragmentScenario的简化方式）
        FragmentScenario<MyFragment> scenario = FragmentScenario.launchInContainer(MyFragment.class);
        
        // 在回调中获取Fragment实例并验证
        scenario.onFragment(fragmentInstance -> {
            // 验证View已创建
            assertThat(fragment.getView()).isNotNull();
          
            // 验证子View状态
        		TextView title = fragment.getView().findViewById(R.id.title);
        		assertThat(title.getText().toString()).isEqualTo("预期文本");
        });
    }
}
```

##### View 交互测试

```java
@RunWith(RobolectricTestRunner.class)
public class ViewInteractionTest {

    @Test
    public void testButtonClick() {
        MainActivity activity = Robolectric.buildActivity(MainActivity.class).create().get();
        
        Button button = activity.findViewById(R.id.submit_button);
        EditText editText = activity.findViewById(R.id.input_field);
        
        // 模拟输入
        editText.setText("test input");
        
        // 模拟点击
        button.performClick();
        
        // 验证结果
        TextView resultView = activity.findViewById(R.id.result_text);
        assertThat(resultView.getText().toString()).isEqualTo("test input");
    }
}
```

##### Shadow 对象使用

常用 Shadow 类：

+ ShadowActivity：验证 Intent 跳转、获取结果。
+ ShadowApplication：验证 Application 行为。
+ ShadowIntent：验证 Intent 内容。
+ ShadowBitmap：模拟图片操作。
+ ShadowLooper：控制 Handler/Looper。
+ ShadowToast：验证 Toast 显示。
+ ShadowAlertDialog：验证 Dialog 弹出。

```java
// 验证Intent跳转
@Test
public void testIntentNavigation() {
    // 1.启动Activity
    MainActivity activity = Robolectric.buildActivity(MainActivity.class).create().get();
    
    // 2.模拟点击按钮，触发startActivity()跳转
    activity.findViewById(R.id.next_button).performClick();
    
    // 3.通过Shadow获取刚才发出的Intent
    // shadowOf()是Robolectric提供的工具方法，返回Activity对应的ShadowActivity
    // getNextStartedActivity()返回最近一次startActivity()传入的Intent
    ShadowActivity shadowActivity = shadowOf(activity);
    Intent nextIntent = shadowActivity.getNextStartedActivity();
    
    // 4.验证跳转目标是否正确
    assertThat(nextIntent.getComponent().getClassName())
        .isEqualTo("com.example.myapp.DetailActivity");
}

// 模拟系统服务
@Test
public void testNetworkState() {
    // 获取ShadowConnectivityManager
    ConnectivityManager connectivityManager = 
        (ConnectivityManager) ApplicationProvider.getApplicationContext().getSystemService(Context.CONNECTIVITY_SERVICE);
    
    ShadowConnectivityManager shadowConnectivityManager = shadowOf(connectivityManager);
    
    // 模拟无网络状态
    shadowConnectivityManager.setActiveNetworkInfo(null);
    
    // 验证你的网络判断逻辑
    assertThat(NetworkUtils.isNetworkAvailable()).isFalse();
}
```

##### 异步代码测试

```java
@RunWith(RobolectricTestRunner.class)
public class AsyncTest {

    @Test
    public void testDelayedTask() {
        // 创建Handler
        Handler handler = new Handler(Looper.getMainLooper());
        
        final boolean[] executed = {false};
        handler.postDelayed(() -> executed[0] = true, 5000);
        
        // 快进时间（毫秒）
        ShadowLooper.runUiThreadTasksIncludingDelayedTasks();
        // 或者快进特定时间
        ShadowLooper.shadowMainLooper().idleFor(5000, TimeUnit.MILLISECONDS);
        
        assertThat(executed[0]).isTrue();
    }
}
```

#### 不能 Mock 吗？为什么需要 Robolectric

先举个例子，同一个代码用纯 Mockito 和用 Robolectric 的区别： 

```java
public class NetworkInfoHelper {

    public static String getNetworkDescription(Context context) {
        // 1.获取系统服务ConnectivityManager
        ConnectivityManager cm = (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE);
        if (cm == null) return "未知网络";

        // 2.获取当前活动的网络
        Network activeNetwork = cm.getActiveNetwork();
        if (activeNetwork == null) {
            return "无网络连接";
        }

        // 3.获取网络能力（判断是Wi-Fi还是蜂窝网络）
        NetworkCapabilities capabilities = cm.getNetworkCapabilities(activeNetwork);
        if (capabilities == null) return "未知网络";

        if (capabilities.hasTransport(NetworkCapabilities.TRANSPORT_WIFI)) {
            // 4.需要读取应用内置的字符串资源
            return context.getString(R.string.network_wifi_connected); 
        } else if (capabilities.hasTransport(NetworkCapabilities.TRANSPORT_CELLULAR)) {
            return context.getString(R.string.network_cellular_connected);
        }

        return "其他网络";
    }
}
```

Mockito：

```java
@Test
public void testWifiConnected() {
    // 1.Mock Context
    Context mockContext = mock(Context.class);
    
    // 2.Mock ConnectivityManager
    ConnectivityManager mockCm = mock(ConnectivityManager.class);
    when(mockContext.getSystemService(Context.CONNECTIVITY_SERVICE)).thenReturn(mockCm);
    
    // 3.Mock Network
    Network mockNetwork = mock(Network.class);
    when(mockCm.getActiveNetwork()).thenReturn(mockNetwork);

    // 4.Mock NetworkCapabilities
    NetworkCapabilities mockCapabilities = mock(NetworkCapabilities.class);
    when(mockCm.getNetworkCapabilities(mockNetwork)).thenReturn(mockCapabilities);
    when(mockCapabilities.hasTransport(NetworkCapabilities.TRANSPORT_WIFI)).thenReturn(true);

    // 5.Mock Resources
    Resources mockResources = mock(Resources.class);
    when(mockContext.getResources()).thenReturn(mockResources);
    when(mockResources.getString(R.string.network_wifi_connected)).thenReturn("已连接无线网");
    when(mockContext.getString(R.string.network_wifi_connected)).thenReturn("已连接无线网");

    // --- 终于可以断言了 ---
    String result = NetworkInfoHelper.getNetworkDescription(mockContext);
    assertEquals("已连接无线网", result);
}
```

Robolectric：

```java
@RunWith(RobolectricTestRunner.class)
@Config(sdk = 30)
public class NetworkInfoHelperTest {

    @Test
    public void testWifiConnected() {
        // 直接获取一个真实的、Robolectric模拟出来的Application Context
        Context context = RuntimeEnvironment.getApplication();

        // 模拟系统状态：比如你想让当前网络变成Wi-Fi
        // Robolectric提供了Shadows机制，可以直接操控系统的状态
        ConnectivityManager cm = (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE);
        
        // 你可以通过Robolectric的Shadow轻松设置当前系统的网络状态、Wi-Fi连接等
        // 或者直接利用它内置的资源解析能力，读取res/values/strings.xml
        String expectedString = context.getString(R.string.network_wifi_connected);
        
        // ...
    }
}
```

可以看到，使用 Mockito 的话，代码会极度臃肿，你的测试代码有 90% 都在写 `when(...).thenReturn(...)`。业务逻辑才 1 行，Mock 逻辑写了 20 行。

那什么时候用 Mockito、什么时候用 Robolectric 呢？ 决策树：

```
测试什么？
│
├─ 纯业务逻辑（不依赖 Android）
│  └─ 用 Mockito
│
├─ 需要 Android 组件（Activity/Fragment/Context/View）
│  └─ 用 Robolectric
│     └─ 如果有非 Android 依赖，配合 Mockito 一起用
│
└─ 需要真实 UI 交互（触摸、动画、渲染）
   └─ 用 Espresso（在真机/模拟器上）
```







