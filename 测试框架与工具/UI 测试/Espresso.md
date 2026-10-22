#### 概述

> https://developer.android.com/training/testing/espresso?hl=zh-cn

Espresso 是 Google 官方为 Android 提供的 UI 自动化测试框架，主要用于测试应用的用户界面交互，提供了简洁易读的 API 来编写可靠的 UI 测试。

核心特点：

+ 自动同步机制：Espresso 通过监听主线程 MessageQueue、AsyncTask 线程池、动画系统，以及自定义 IdlingResource 四大维度，在每次操作前轮询等待所有异步任务完成，才将测试动作注入主线程执行。在没有 Espresso 之前，开发者需要手动处理等待逻辑，测试既不稳定又难维护。

  ```java
  // 旧方式：手动等待
  onView(withId(R.id.button)).perform(click());
  // 问题1：不等待会失败（UI 还没更新）
  // 问题2：等待时间不确定（网络慢时 2 秒可能不够，快时又浪费时间）
  // 问题3：测试不稳定（有时通过，有时失败）
  Thread.sleep(2000);
  onView(withId(R.id.textView)).check(matches(withText("加载完成")));
  
  
  // Espresso方式：自动等待
  onView(withId(R.id.button)).perform(click());
  // Espresso 会自动等待：
  // 1. 主线程消息队列清空
  // 2. 网络请求完成（如果注册了 IdlingResource）
  // 3. UI 渲染完毕
  // 然后才执行断言，测试稳定可靠
  onView(withId(R.id.textView)).check(matches(isDisplayed()));
  ```

+ 简洁的链式 API：不需要在写繁琐的 Instrumentation API，提高可读性和可维护性。

  ```java
  // Instrumentation API
  activityRule.getScenario().onActivity(activity -> {
      activity.runOnUiThread(() -> {
          TextView textView = activity.findViewById(R.id.textView);
          textView.setText("Hello");
      });
  });
    
  // Espresso方式
  onView(withId(R.id.textView)).check(matches(withText("Hello")));
  ```

#### 测试目录结构

```
app/
├── src/
│   ├── main/
│   │   ├── java/com/example/app/
│   │   │   └── MainActivity.java
│   │   ├── res/
│   │   └── AndroidManifest.xml
│   ├── test/
│   │   └── java/com/example/app/
│   │       └── ExampleUnitTest.java
│   └── androidTest/
│       └── java/com/example/app/
│           └── ExampleInstrumentedTest.java
```

- main：存放应用的主代码和资源，会被打包到最终的 APK 中，运行在真实设备或模拟器上。
- test：存放本地单元测试代码，在 JVM 上运行（不需要设备），用于测试不依赖 Android API 的纯逻辑代码。
- androidTest：存放仪器化测试代码（包括 UI 测试和集成测试），需在真实设备或模拟器上运行，可以访问完整的 Android API。

#### 基础使用

##### 基本测试结构

```groovy
androidTestImplementation 'androidx.test.espresso:espresso-core:3.5.1'
androidTestImplementation 'androidx.test:runner:1.5.2'
androidTestImplementation 'androidx.test:rules:1.5.2'
```

```java
@RunWith(AndroidJUnit4.class)
public class LoginActivityTest {

    @Rule
    // 自动启动Activity
    public ActivityScenarioRule<LoginActivity> activityRule = new ActivityScenarioRule<>(LoginActivity.class);
    // 等同于在@Test里这么写
  	// @Test
    // public void loginSuccess() {
    //     ActivityScenario<LoginActivity> scenario = ActivityScenario.launch(LoginActivity.class);
		//     onView(withId(R.id.username_edit_text)).perform(typeText("testuser"));
    //     scenario.close();  // 手动关闭
    // }

    @Test
    public void loginSuccess() {
        // 输入用户名
        onView(withId(R.id.username_edit_text))
            .perform(typeText("testuser"), closeSoftKeyboard());

        // 输入密码
        onView(withId(R.id.password_edit_text))
            .perform(typeText("password123"), closeSoftKeyboard());

        // 点击登录按钮
        onView(withId(R.id.login_button))
            .perform(click());

        // 验证结果
        onView(withText("Welcome"))
            .check(matches(isDisplayed()));
    }
}
```

运行命令：

```groovy
// 不仅仅是Espresso，所有androidTest目录中的测试
./gradlew connectedAndroidTest
```

查看测试结果：

```groovy
open app/build/reports/androidTests/connected/debug/index.html
```

##### 动画因素

动画可能会持续触发 `requestLayout()` 或 `invalidate()`，导致主线程 MessageQueue 持续有消息要处理，最终会导致 Espresso 等待超时。

```groovy
android {
    testOptions {
        animationsDisabled = true
    }
}
```

##### 核心组件

1）ViewMatchers - 查找视图

```java
onView(withId(R.id.xxx))           // 按资源 ID 匹配
onView(withText("xxx"))            // 按文本内容匹配
```

2）ViewActions - 执行操作

```java
.perform(click())                          // 点击
.perform(typeText("xxx"))                  // 输入文本
.perform(clearText())                      // 清除文本
.perform(pressBack())                      // 按返回键
.perform(swipeLeft())                      // 左滑
```

3）ViewAssertions - 验证状态

```java
.check(matches(isDisplayed()))             // 验证可见
.check(matches(withText("xxx")))           // 验证文本
.check(doesNotExist())                     // 验证不存在
```

#### 自动同步机制

Espresso 之所以能告别脆弱的 `Thread.sleep()`，核心在于其与 Android 主线程事件循环（Looper/MessageQueue）绑定的空闲检测机制。在执行任何 UI 操作（如 `onView(...).perform(click())`）或断言（`check(matches(...))`）之前，Espresso 会确保所有被监控的异步系统均处于空闲状态（Idle，即执行完毕），测试操作才会真正执行。

空闲监测机制：

```
onView().perform() / onView().check()
       │
       ▼
UiController.loopMainThreadUntilIdle()
       │
       ▼
  ┌─────────────────────────────────────┐
  │      同步检查（必须全部通过）           │
  └─────────────────────────────────────┘
       │
       ├─► 1. 所有IdlingResource是否空闲？
       │      └─ 否 ──► 挂起等待，直到回调onTransitionToIdle()
       │         （对于外部异步框架，比如RxJava/网络请求等，Android运行时无法凭空知道它们何时结束，需通过此机制手动告诉Espresso）
       │
       ├─► 2. AsyncTask 线程池是否有运行中的任务？
       │      └─ 是 ──► 等待任务完成
       │         （防止MessageQueue看似空闲，实际AsyncTask 完成后又会post新消息到主线程）
       │
       └─► 3. 主线程MessageQueue是否有待处理消息？
              └─ 是 ──► 驱动主线程Looper处理消息，直到队列空闲
       │
       ▼ (所有条件满足，系统进入Idle状态)
       │
  在主线程执行操作：
  ├─ perform() → 注入MotionEvent（点击、滑动等）
  └─ check()   → 进行View属性断言
```

但主线程消息队列为空并不代表绘制会立即执行。在 Android 中，View 的 `requestLayout()` 或 `invalidate()` 并不会立即执行绘制，而是向主线程发送一个请求，等待下一个 VSYNC 信号（通常按 60Hz/120Hz 周期性到达，约每 16ms 或 8ms 一次）。Espresso 的内部不仅检查消息队列，还会与 Choreographer 机制同步。当某个 View 发起绘制请求后，Espresso 会等待 Traversal 流程（measure → layout → draw）真正执行完毕，确保验证状态时已执行完毕。

#### 进阶功能

##### AdapterView 测试

```groovy
androidTestImplementation 'androidx.test.espresso:espresso-contrib:3.5.1'
```

https://developer.android.google.cn/training/testing/espresso/lists?hl=zh-cn

##### Intent 测试

```groovy
androidTestImplementation 'androidx.test.espresso:espresso-intents:3.5.1'
```

https://developer.android.google.cn/training/testing/espresso/intents?hl=zh-cn

##### WebView 测试

```groovy
androidTestImplementation 'androidx.test.espresso:espresso-web:3.5.1'
```

https://developer.android.google.cn/training/testing/espresso/web?hl=zh-cn

##### 异步任务处理

```groovy
androidTestImplementation 'androidx.test.espresso:espresso-idling-resource:3.5.1'
```

https://developer.android.google.cn/training/testing/espresso/idling-resource?hl=zh-cn



