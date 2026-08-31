Overlay 

原生 Android 里，**Dialog 是一类组件，Overlay 是一种窗口能力**。系统没有叫 `Overlay` 的类，和 `android.app.Dialog` 不是对等物。

## Dialog：有现成类的弹窗

`android.app.Dialog` / `DialogFragment` 是框架提供的 UI 组件。本质是：在当前 Activity 上面再开一个 **Window**。

常见特征：

- 绑在当前 Activity 上，用的是应用窗口类型（如 `TYPE_APPLICATION`、`TYPE_APPLICATION_ATTACHED_DIALOG`）
- 一般是**模态**：挡住下面页面，要点按钮或返回才关
- 生命周期跟 Activity 走：页面 finish，Dialog 一起没
- 默认会有蒙层（dim），点外部可关（可配）
- 不需要特殊权限

```java
new AlertDialog.Builder(activity)
    .setTitle("确认")
    .setMessage("要关闭吗？")
    .setPositiveButton("确定", null)
    .show();
```

`DialogFragment` 只是把这套窗口塞进 Fragment 生命周期里，方便旋转、状态恢复。

## Overlay：浮在别的界面上的窗口

Overlay 不是某个类，而是用 `WindowManager.addView()` 加一层窗口，让它浮在别的内容上面。系统把它叫 **system overlay / application overlay**。

API 26 之后标准类型是 `TYPE_APPLICATION_OVERLAY`，需要权限：

```xml
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
```

用户还要在系统设置里打开「显示在其他应用的上层」。

常见特征：

- **不依赖某个 Activity**，Service、SystemUI 都能加
- 可以盖在别的 App 上面（悬浮球、录屏控件、来电浮层）
- 自己管显示/隐藏，Activity 关了它还可以在
- 不一定模态：可以 `FLAG_NOT_FOCUSABLE`（不抢焦点）、`FLAG_NOT_TOUCH_MODAL`（点外面穿透）
- 通常没有系统自带的标题栏、按钮模板，View 全自己画

```java
WindowManager.LayoutParams lp = new WindowManager.LayoutParams(
    WRAP_CONTENT, WRAP_CONTENT,
    WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,
    WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE,
    PixelFormat.TRANSLUCENT
);
windowManager.addView(view, lp);
```

## 对比

| | Dialog | Overlay（系统浮窗） |
|---|---|---|
| 是什么 | `Dialog` / `DialogFragment` | 一种 Window 类型 + `WindowManager` |
| 挂在哪 | 当前 Activity | 系统窗口层，可不绑页面 |
| 能盖别的 App 吗 | 不能（普通用法） | 能 |
| 权限 | 不需要 | `SYSTEM_ALERT_WINDOW` |
| 生命周期 | 跟 Activity | 自己管 |
| 交互 | 通常模态、抢焦点 | 可模态也可穿透 |
| UI | 有模板（AlertDialog 等） | 没有，自己 inflate |

窗口层级大致是：

```
系统层 Overlay（TYPE_APPLICATION_OVERLAY）  ← 最高，可跨应用
    └── 当前 App 的 Dialog 窗口
            └── Activity 窗口
```

所以：Dialog 是「这个页面上的弹窗」；Overlay 是「整块屏幕上再盖一层」。

## 和你们 PATAC 那套怎么对应

车机上的 `TYPE_PATAC_OVERLAY` / `TYPE_PATAC_SIMPLEDIALOG` 是厂商自己加的窗口类型，AOSP 没有这两个名字。产品语义接近：

- **`PatacDialog`** ≈ 原生 Dialog：确认框，模态，点外部不关
- **`PatacOverLay*`** ≈ 用 Dialog/DialogFragment **模拟** Overlay：更轻、可点外部关，窗口 type 走 overlay
- **`PatacWithServiceOverLay`** 最接近原生 Overlay：直接 `Dialog` + overlay type，不绑 Fragment，Service 也能弹

要注意：`PatacOverLayDialog` 虽然叫 Overlay，底座仍是 `DialogFragment`，不是 `WindowManager.addView()`。叫 Overlay 主要是 **交互和窗口层级** 像浮层，不是 Android 官方那个跨应用 Overlay API。

## 原生里容易混的几个

- **`PopupWindow`**：更轻的浮层，挂在某个 View 旁边（菜单、下拉），不是 Dialog，也不是系统 Overlay
- **`Toast`**：系统短提示，不能点
- **Theme Overlay**：只是主题叠加，和弹窗无关
- **`SYSTEM_ALERT_WINDOW` Overlay**：才是上面说的跨应用浮窗

一句话：**Dialog 是组件；Overlay 是窗口层。** Dialog 活在当前页面里；Overlay 活在窗口栈更上面，甚至可以活在所有 App 上面。