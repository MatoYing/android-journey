#### 概述

UI Automator 是 Google 提供的跨应用 UI 测试框架，基于 Android 无障碍服务。与 Espresso 局限于单个 App 不同，它可以操作设备上的任意界面——包括系统设置、通知栏、其他 App，因此常用于跨进程/跨应用的端到端测试。

核心 API 结构:

```groovy
UiDevice          ← 设备级入口（点击Home、旋转屏幕、截图）
  ├── UiSelector       ← 定位条件构建器（文本、ID、类名）
  ├── UiObject         ← 单个UI元素的操作代理（click/setText）
  ├── UiCollection     ← 元素集合
  └── UiWatcher        ← 弹窗/中断监听器
```

+ UiDevice：手机本身的操作入口，按 Home 键、旋转屏幕、截图等，不操作某个具体按钮，而是操作整台设备。

  ```java
  device.pressHome();   // 按Home键回桌面
  ```

+ UiSelector：描述"我要找谁"的条件，比如按文本、ID、类名来定位元素，本身不执行操作。

  ```java
  new UiSelector().text("确定")  // 找写着"确定"的按钮
  ```

+ UiObject：找到元素后，真正去操作它的那只手，负责点击、输入、长按等具体动作。

  ```java
  device.findObject(new UiSelector().text("确定")).click();
  ```

+ UiCollection：管理一组同类型元素，比如列表里的所有商品卡片，可以获取数量、按索引取元素。

  ```java
  UiCollection items = new UiCollection(new UiSelector().className("TextView"));
  int count = items.getChildCount();  // 一共有几个
  ```

+ UiWatcher：自动处理意外弹窗的监听器，测试过程中出现意料之外的对话框时自动帮你关掉。

  ```java
  device.registerWatcher("弹窗", (watcher) -> {
      UiObject allow = device.findObject(new UiSelector().text("允许"));
      if (allow.exists()) { allow.click(); return true; }
      return false;
  });
  ```

#### 基础使用：

##### 1）依赖

```groovy
androidTestImplementation 'androidx.test.uiautomator:uiautomator:2.3.0'
```

##### 2）获取 UiDevice

```java
@RunWith(AndroidJUnit4.class)
public class LauncherTest {

    private UiDevice device;

    @Before
    public void setup() {
        Instrumentation instrumentation = InstrumentationRegistry.getInstrumentation();
        device = UiDevice.getInstance(instrumentation);
    }
}
```

##### 3）定位并操作元素

```java
// 通过文本查找并点击
UiObject settingsButton = device.findObject(new UiSelector().text("Settings"));
settingsButton.click();

// 通过resource-id查找
UiObject input = device.findObject(
    new UiSelector().resourceId("com.example:id/edit_text")
);
input.setText("hello");

// 通过类名+索引
UiObject secondButton = device.findObject(
    new UiSelector().className(Button.class).index(1)
);
```

##### 4）等待元素出现

```java
// 设置最多等5秒，元素出现即返回
UiObject target = device.wait(
    Until.findObject(By.text("Welcome")),
    5000
);
assertNotNull(target);
```

##### 5）设备级操作

```java
device.pressHome();                // 按Home键
device.pressBack();                // 按返回键
device.setOrientationLeft();       // 强制横屏
device.takeScreenshot(new File("/sdcard/screen.png"));  // 截图
device.swipe(500, 1500, 500, 500, 20);  // 滑动（下拉/翻页）
device.wakeUp();                   // 唤醒屏幕
```

##### 6）启动其他应用

```java
// 通过shell命令启动任意App
device.executeShellCommand("am start -n com.example/.MainActivity");

// 通过Context启动
Context context = InstrumentationRegistry.getInstrumentation()
    .getTargetContext();
Intent intent = context.getPackageManager()
    .getLaunchIntentForPackage("com.other.app");
context.startActivity(intent);
```

##### 7）处理弹窗（UiWatcher）

```java
device.registerWatcher("permission_dialog", (watcher) -> {
    UiObject allow = device.findObject(new UiSelector().text("允许"));
    if (allow.exists()) {
        allow.click();
        return true;  // 表示已处理
    }
    return false;
});
device.runWatchers();   // 触发检查
```

#### 原理

UI Automator 本质是一个无障碍服务（AccessibilityService）。它通过 UiAutomation 向 AccessibilityManagerService（系统服务）注册自己为无障碍服务，借助 Binder IPC 获得三项能力：

1. 遍历任意窗口的 AccessibilityNodeInfo 树 —— 感知 UI
2. 执行点击、滑动等手势 —— 操作 UI
3. 监听窗口变化事件（打开、关闭、内容变化）—— 响应 UI 变化

这三项能力都是通过 AccessibilityService 的权限通道获得的，和普通的无障碍服务原理一样，只是由系统内置的测试服务来承载。

#### 与 Espresso 差异

两者在 App 内 UI 操作上存在重叠，但定位不同：

- Espresso：专注 App 内部 UI 测试，速度快（同进程，不需要 IPC）、定位精准（因为是和应用同进程，可以直接拿到 View 的所有属性；而 UI Automator 和应用不是同一个进程，只能通过无障碍服务暴露出来的信息来执行相应逻辑）、自动同步主线程。
- UI Automator：专注 App 外部操作，如系统弹窗、通知栏、跨应用跳转。

实际使用中，通常以 Espresso 为主，只在涉及系统级或跨应用的部分才引入 UI Automator 补充。
