### 补间动画

补间动画（需要指定动画的“开始”状态和“结束”状态，系统会自动帮你计算出“中间”的所有过渡状态）是针对 View 的视觉效果进行变换的动画，它包含四种基本类型：平移（Translate）、缩放（Scale）、旋转（Rotate）和透明度（Alpha）。它的特点是**只改变了 View 的视觉绘制位置，而不改变 View 的实际属性（如点击事件的位置依然在原处）**。

- R.anim.xxx，为 Animation (补间动画)。
- R.animator.xxx，为 Animator (属性动画)。

#### 1）平移动画

平移动画控制 View 在水平（X）或垂直（Y）方向上的移动。

```java
// 参数含义：
// fromXDelta: 起始X坐标
// toXDelta: 结束X坐标
// fromYDelta: 起始Y坐标
// toYDelta: 结束Y坐标
// Animation.RELATIVE_TO_SELF 表示相对于自身百分比，0.5f 代表 50%
TranslateAnimation translateAnim = new TranslateAnimation(
    Animation.RELATIVE_TO_SELF, 0f,    // 起始X：自身左侧
    Animation.RELATIVE_TO_SELF, 1.0f,  // 结束X：向右移动一个自身的宽度
    Animation.RELATIVE_TO_SELF, 0f,    // 起始Y
    Animation.RELATIVE_TO_SELF, 0f     // 结束Y
);

translateAnim.setDuration(800);
translateAnim.setInterpolator(new AccelerateDecelerateInterpolator()); // 设置插值器（先加速后减速）
view.startAnimation(translateAnim);
```

#### 2）缩放动画

缩放动画控制 View 的大小变化，需要指定缩放的中心点（轴心）。

```java
// 参数含义：
// fromX, toX: X轴缩放比例（0.0 - n）
// fromY, toY: Y轴缩放比例
// pivotXType, pivotXValue: X轴中心点类型和值
// pivotYType, pivotYValue: Y轴中心点类型和值
ScaleAnimation scaleAnim = new ScaleAnimation(
    1.0f, 2.0f, 1.0f, 2.0f,             // 从1倍放大到2倍
    Animation.RELATIVE_TO_SELF, 0.5f,   // X轴中心点在 View 正中间
    Animation.RELATIVE_TO_SELF, 0.5f    // Y轴中心点在 View 正中间
);

scaleAnim.setDuration(1000);
view.startAnimation(scaleAnim);
```

#### 3）旋转动画

旋转动画控制 View 绕某个中心点进行角度旋转。

```java
// 参数含义：
// fromDegrees: 起始角度
// toDegrees: 结束角度
// pivotXType, pivotXValue: 旋转中心点X
// pivotYType, pivotYValue: 旋转中心点Y
RotateAnimation rotateAnim = new RotateAnimation(
    0f, 360f,                           // 旋转一整圈
    Animation.RELATIVE_TO_SELF, 0.5f, 
    Animation.RELATIVE_TO_SELF, 0.5f
);

rotateAnim.setDuration(1000);
rotateAnim.setRepeatCount(2); // 重复次数（此处会总共运行3次）
view.startAnimation(rotateAnim);
```

#### 4）透明度动画

透明度动画控制 View 的透明度变化，范围从 0.0（完全透明）到 1.0（完全不透明）。

```java
// 参数：fromAlpha（起始透明度），toAlpha（结束透明度）
AlphaAnimation alphaAnim = new AlphaAnimation(1.0f, 0.0f); 

alphaAnim.setDuration(1000); // 设置动画持续时间（毫秒）
alphaAnim.setFillAfter(true); // 动画结束后停留在结束状态

// 执行动画
view.startAnimation(alphaAnim);
```

#### 5）组合动画

```java
// 创建动画集合，参数shareInterpolator表示是否共用一个插值器
AnimationSet animSet = new AnimationSet(true);

// 1.创建缩放动画
ScaleAnimation scale = new ScaleAnimation(1.0f, 0.5f, 1.0f, 0.5f, 
    Animation.RELATIVE_TO_SELF, 0.5f, Animation.RELATIVE_TO_SELF, 0.5f);

// 2.创建透明度动画
AlphaAnimation alpha = new AlphaAnimation(1.0f, 0.3f);

// 将动画添加到集合中
animSet.addAnimation(scale);
animSet.addAnimation(alpha);

animSet.setDuration(1500);
animSet.setFillAfter(true);

// 执行组合动画
view.startAnimation(animSet);
```

如果效果是固定的，其实使用 xml 方式更好。

```xml
<!-- set相当于AnimationSet -->
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="1000"
    android:fillAfter="true"
    android:interpolator="@android:anim/accelerate_decelerate_interpolator">

    <!-- 透明度 -->
    <alpha
        android:fromAlpha="1.0"
        android:toAlpha="0.5" />

    <!-- 缩放 -->
    <scale
        android:fromXScale="1.0"
        android:toXScale="1.5"
        android:fromYScale="1.0"
        android:toYScale="1.5"
        android:pivotX="50%"
        android:pivotY="50%" />

    <!-- 平移 -->
    <translate
        android:fromXDelta="0"
        android:toXDelta="100"
        android:fromYDelta="0"
        android:toYDelta="100" />

    <!-- 旋转 -->
    <rotate
        android:fromDegrees="0"
        android:toDegrees="360"
        android:pivotX="50%"
        android:pivotY="50%" />

</set>
```

```java
// 代码中绑定执行
// 1.使用AnimationUtils加载动画资源
Animation myAnim = AnimationUtils.loadAnimation(getContext(), R.anim.view_multi_animation);
// 2.View执行动画
view.startAnimation(myAnim);
```

#### 动画监听器

可以通过实现 AnimationListener 的方式，为动画添加监听器。

```java
// 1. 创建动画（以透明度动画为例）
AlphaAnimation alphaAnim = new AlphaAnimation(0.0f, 1.0f);
alphaAnim.setDuration(300);

// 2. 设置动画监听器
alphaAnim.setAnimationListener(new Animation.AnimationListener() {
    @Override
    public void onAnimationStart(Animation animation) {
        // 动画开始时触发
        // 场景举例：可以在此时禁用按钮，防止用户多次点击
        Log.d("Animation", "动画开始了");
    }

    @Override
    public void onAnimationEnd(Animation animation) {
        // 动画结束时触发
        // 场景举例：动画结束后跳转Activity，或者把View的Visibility设为GONE
        Log.d("Animation", "动画结束了");
    }

    @Override
    public void onAnimationRepeat(Animation animation) {
        // 动画重复执行时触发（如果设置了setRepeatCount）
        Log.d("Animation", "动画重复了");
    }
});

// 3. 启动动画
button.startAnimation(alphaAnim);
```

#### 自定义动画

自定义 View 动画的核心是矩阵变换。平移、缩放这些看似完全不同的操作，在数学上都可以统一用一个 3x3 的矩阵来表示。比如想让一个 View 先旋转 30 度，再平移 100 像素，最后放大 2 倍，你不需要分三步去计算坐标。你只需要把“旋转矩阵”、“平移矩阵”和“缩放矩阵”相乘，得到一个最终的“结果矩阵”，计算机只需要拿着这个结果矩阵去处理每一个像素点即可。

```java
public class SlideAndScaleAnimation extends Animation {
    @Override
    protected void applyTransformation(float interpolatedTime, Transformation t) {
        // interpolatedTime：0 → 1，由TimeInterpolator控制节奏
        Matrix matrix = t.getMatrix();
        
        // 平移：从0滑到300px
        matrix.postTranslate(300 * interpolatedTime, 0);
  
        // 缩放：从1.0放大到2.0
        float scale = 1.0f + interpolatedTime;
        matrix.postScale(scale, scale);
    }
}

SlideAndScaleAnimation anim = new SlideAndScaleAnimation();
anim.setDuration(1000);
anim.setInterpolator(new AccelerateInterpolator());
view.startAnimation(anim);
```

由于其底层涉及较深的线性代数逻辑且实现繁琐，基本不使用（特殊的 3D 旋转效果里会用到），这里不做介绍。

#### 为什么点击事件不会跟着走

补间动画的工作原理，通过 Matrix 变换 改变了 Canvas 的绘制表现，而非 View 属性。

```java
// 在父容器绘制子View的过程中
void drawChild(Canvas canvas, View child) {
    // 备份当前画布状态
    canvas.save(); 

    // 将动画产生的位移/旋转矩阵应用到画布上
    // 这里改变的是画布的坐标系，而不是View的坐标属性
    canvas.concat(animationMatrix); 

    // 在“变了形”的画布上绘制子View
    // 如果Canvas没变形，draw(0, 0)就落在屏幕(0, 0)
    // 如果Canvas应用了Matrix.postTranslate(500, 0)，Canvas的内部逻辑就变成了，你以为你在画x = 0，Canvas实际渲染x= 0 + 500
    // 子View依然认为自己在(left, top)处绘制，但画出来后视觉上已经偏离了
    child.draw(canvas); 

    // 恢复画布
    canvas.restore(); 
}
```

在这个过程中，View 的 left、top、right、bottom 等属性完全没有变化，而点击事件的判断逻辑还是基于坐标的。所以补间动画改变的是“绘制效果”，而属性动画改变的是“对象本身”。

#### 动画的播放时机

这个概念并不是针对“补间动画”，是所有的动画。也包括帧动画，因为它需要通过 View 作为载体。

onStart（可见）与 onResume（可交互）描述的 Window（系统窗口），而非 View（视图）的内容。这也解释了为什么在冷启动应用时，如果你的 View 很复杂、渲染很慢，你会先看到一瞬间的白屏或黑屏（取决于你的 Theme 背景色），然后才看到真实的按钮和文字。

下面是正确的执行时序，可以看到播放动画最好的地方是 `onWindowFocusChanged()` 中启动，或者使用 post 方法。

```
执行顺序：
onCreate() / onStart() / onResume() —— Window就绪
Measure -> Layout -> Draw —— View树真正完成渲染（此时才有宽高）
View.post(Runnable) —— 任务被投递到队列末尾，保证View的测量和布局已经完成后执行
onWindowFocusChanged() —— 窗口里所有的一切都绘制完成、获取了焦点，此时View是可见且可操作的
```

### 帧动画

帧动画其原理类似于“翻书页”，通过依次播放一系列预先准备好的图片，利用人眼的视觉暂留效应形成连续动画。适用于比较复杂的图像变换，比如任务走动这种。

```xml
<!-- oneshot为true表示只播放一次，false表示循环播放 -->
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot="false">
    
    <!-- duration为每张图片的显示时长（毫秒） -->
    <item android:drawable="@drawable/frame1" android:duration="100" />
    <item android:drawable="@drawable/frame2" android:duration="100" />
    <item android:drawable="@drawable/frame3" android:duration="100" />
</animation-list>
```

```java
ImageView imageView = findViewById(R.id.iv_loading);
// 1. 设置背景或图片资源
imageView.setImageResource(R.drawable.loading_anim);

// 2. 获取动画对象
AnimationDrawable frameAnim = (AnimationDrawable) imageView.getDrawable();

// 3. 启动动画
frameAnim.start();

// 停止动画
// frameAnim.stop();
```

首先有几个概念需要明确：

+ 刷新率：**屏幕每秒换了多少张（刷新，只是显示）**，即把收到的画面显示出来，它是固定的，由硬件决定。现在的手机屏幕通常是 60Hz（每 16.6ms 刷新一次）或 120Hz（每 8.3ms 刷新一次）。这代表屏幕**每秒钟会重新画 60 次或 120 次**。

+ 帧率（FPS）：**GPU 每秒画了多少张（渲染，需要计算）**，需要计算像素、合成图层、产出画面，频率是由 App 决定快慢。电影标准为 24FPS（人类肉眼在 24 帧左右就能感觉到动作是连续的了），游戏标准为 60FPS。

+ 帧动画时长：比如 `android:duration="100"`，意思是，这张图片在屏幕上会停够 100 毫秒，再换下一张。比如屏幕是 60Hz，屏幕在刷新 6 次 的时间里，显示的其实都是同一张图片。直到第 7 次刷新（大约 116ms 时），才会显示下一张。

比如需求会告诉你这个动画要播 2 秒，帧率是 30FPS。那么 UI 需要给我们提供 `帧率 30 × 总时长 2s = 60 张图`；`每帧时长 = 1000ms / 帧率 30 = 33.3ms`。

它的一个问题是，由于它需要一次性将所有图片加载到内存中，如果图片过多或单张图片过大可能会引发 **OOM**。所以用帧动画时，和需求都会确认好这个帧动画的帧率，不能太高，太高意味着需要的图片越多。另外图片不能太清晰，太清晰意味着单张图片体积越大。

### 属性动画

补间动画改变的是“绘制效果”，并没有改变视图的真实属性，而属性动画改变的是“对象本身”，也就是只要对象有这个属性，它就能实现动画效果。

下面先介绍三个核心类：

#### 1）ValueAnimator

属性动画的核心引擎。它负责计算动画过程中的值（从开始到结束的平滑过渡），但它不直接作用于对象。需要通过监听器手动更新对象的属性。

对于 ObjectAnimator 简化了这个过程，但该属性必须有 Setter 方法（当目标属性缺少 Setter 方法、或其数值由其他状态计算而来时，可以通过包装类（Wrapper）来补充实现 set 方法供 ObjectAnimator 调用），或者想在动画过程中同时改变多个不相关的属性、加入更其他的逻辑操作，ValueAnimator 更适合。

```java
// 让一个按钮的宽度从100变到500
// 创建ValueAnimator，设定数值范围
ValueAnimator valueAnimator = ValueAnimator.ofInt(100, 500);
valueAnimator.setDuration(1000);
// 添加监听器，每一帧数值更新时都会回调
valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
    @Override
    public void onAnimationUpdate(ValueAnimator animation) {
        // 获取当前帧产生的数值
        int animatedValue = (int) animation.getAnimatedValue();
        
        // 手动将数值应用到View上
        ViewGroup.LayoutParams params = btn.getLayoutParams();
        params.width = animatedValue;
        btn.setLayoutParams(params);
    }
});

// 启动
valueAnimator.start();
```

#### 2）ObjectAnimator（最常用）

它是 ValueAnimator 的子类。它允许你直接指定目标对象和属性名，本质上就是一个**利用反射，不断计算数值并塞给对象的 setter 方法的定时任务**。

```java
// 透明度动画
ObjectAnimator animator = ObjectAnimator.ofFloat(btn, "alpha", 1.0f, 0.0f, 1.0f);
animator.setDuration(2000); // 持续2秒
animator.start();

// 平移动画
ObjectAnimator animator = ObjectAnimator.ofFloat(btn, "translationX", 0f, 300f);
animator.setDuration(1000);
animator.start();

// 旋转动画
ObjectAnimator animator = ObjectAnimator.ofFloat(btn, "rotation", 0f, 360f);
animator.setDuration(1000);
animator.start();

// 缩放动画
ObjectAnimator animator = ObjectAnimator.ofFloat(btn, "scaleX", 1f, 2f);
animator.setDuration(1000);
animator.start();
```

另外 xml 也可以，但实际开发中，用的多的还是通过  Java。

```xml
<!-- 新建一个文件res/animator/move_and_rotate.xml -->
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:ordering="together"> <!-- together 同时播放，sequentially 顺序播放 -->

    <objectAnimator
        android:propertyName="translationX"
        android:duration="1000"
        android:valueFrom="0"
        android:valueTo="200"
        android:valueType="floatType" />

    <objectAnimator
        android:propertyName="rotation"
        android:duration="1000"
        android:valueFrom="0"
        android:valueTo="360"
        android:valueType="floatType" />
</set>
```

```java
Animator animator = AnimatorInflater.loadAnimator(context, R.animator.move_and_rotate);
animator.setTarget(btn);
animator.start();
```

另外，为什么要反射？

```java
// 伪代码，如果框架写死了直接调用
if (propertyName.equals("alpha")) {
    target.setAlpha(value);
} else if (propertyName.equals("translationX")) {
    target.setTranslationX(value);
} 
// ... 无限穷举下去

// Android系统自带的View属性有几十个，写死勉强可以。但是，如果您自己写了一个自定义View，怎么办呢？
// 如果不用反射，就必须自己监听每一帧的值，然后手动去set，这其实就是它的父类ValueAnimator 干的活
```

#### 3）AnimatorSet

用于组合多个动画，同时播放、顺序播放或延迟播放。

```java
// 让一个按钮一边移动、一边变透明、一边旋转 
ObjectAnimator move = ObjectAnimator.ofFloat(btn, "translationX", 0f, 300f);
ObjectAnimator rotate = ObjectAnimator.ofFloat(btn, "rotation", 0f, 360f);
ObjectAnimator alpha = ObjectAnimator.ofFloat(btn, "alpha", 1f, 0.5f);

AnimatorSet animSet = new AnimatorSet();
animSet.playTogether(move, rotate, alpha); // 同时播放
// 或者定义顺序：
// animSet.play(move).with(rotate).after(alpha); 
animSet.setDuration(2000);
animSet.start();
```

ValueAnimator  也能操作多个动画，AnimatorSet 更多是做多个动画的编排。

#### ViewPropertyAnimator

ViewPropertyAnimator 是专门为 View 的常见属性（透明度、位移、缩放、旋转）设计的动画辅助类。它通过 `view.animate()` 方法获取，并采用链式调用。

比起 ObjectAnimator，一方面简化了操作，在性能上也有所提升。

ObjectAnimator 默认是通过反射调用相关属性的，而 ViewPropertyAnimator 是专门为 View 的常见属性设计的，不会处理自定义的属性，不用走反射，直接调 set 方法。

另外对于 ObjectAnimator，每一个 ObjectAnimator 都是一个独立的“引擎”。如果你同时改变 Alpha、ScaleX、ScaleY，系统会启动 3 个独立的动画引擎，每个引擎都通过反射去寻找 setAlpha、setScaleX 等方法，这会导致每一帧产生三次反射调用和三次 View 重绘请求。而 ViewPropertyAnimator ，它内部只维护了一个 ValueAnimator，会等待这一帧所有的属性（位移、缩放、透明度等）全部计算完毕后，只向系统请求一次重绘。

```java
myView.animate()
    .translationX(100f)    // 平移到 X=100
    .translationXBy(100f)  // 在当前基础上再平移 100 (带有 By 的是增量)
    .alpha(0f)             // 透明度变为 0
    .rotation(360f)        // 旋转 360 度
    .scaleX(1.5f)          // 横向缩放 1.5 倍
    .setDuration(1000)     // 时长
    .setInterpolator(new AccelerateInterpolator()) // 插值器
    .setStartDelay(200)    // 延迟
    .withStartAction(new Runnable() { // 动画开始时的逻辑
        @Override
        public void run() { /* 逻辑 */ }
    })
    .withEndAction(() -> { // 动画结束时的逻辑（支持 Lambda）
        /* 逻辑 */
    });
```

#### 插值器（Interpolator）与估值器（Evaluator）

1）插值器：控制“速度”，根据时间流逝的百分比，计算出当前属性改变的程度。

+ 输入值：动画已执行的时间占比（0 到 1）。
+ 输出值：当前属性变化的百分比。

常见的系统内置插值器：

- LinearInterpolator：匀速直线运动。
- AccelerateInterpolator：加速运动。
- DecelerateInterpolator：减速运动。
- AccelerateDecelerateInterpolator：先加速再减速（默认值）。

2）估值器：控制“数值”，根据插值器算出来的“百分比”，计算出具体的“属性值”。

+ 输入值：插值器给出的百分比、起始值、结束值。
+ 输出值：具体类型的值（int、float、Color 等）。

常见的系统内置估值器：

+ IntEvaluator：计算整数。
+ FloatEvaluator：计算浮点数。
+ ArgbEvaluator：计算颜色的渐变过渡。

```java
// 让背景色从红色变到蓝色
ObjectAnimator anim = ObjectAnimator.ofInt(view, "backgroundColor", 0xFFFF0000, 0xFF0000FF);
anim.setDuration(3000);
// anim.setInterpolator(new AccelerateDecelerateInterpolator); // 默认
anim.setEvaluator(new ArgbEvaluator());
anim.start();
```

当然插值器、估值器都能自定义，用来实现更复杂的效果。

```java
public class MySquareInterpolator implements TimeInterpolator {
    @Override
    public float getInterpolation(float input) {
        // input是时间流逝率（0.0到1.0）
        // 返回值是动画进度（0.0到1.0）
        
        return input * input; 
    }
}

public class PointEvaluator implements TypeEvaluator<Point> {
    @Override
    public Point evaluate(float fraction, Point startValue, Point endValue) {
        // fraction是插值器传过来的动画进度（0.0到1.0）
        // startValue和endValue是动画的起点和终点
        
        float x = startValue.x + fraction * (endValue.x - startValue.x);
        float y = startValue.y + fraction * (endValue.y - startValue.y);
        
        // 返回当前进度的具体坐标对象
        return new Point(x, y);
    }
}
```

#### 监听动画状态

```java
animator.addListener(new Animator.AnimatorListener() {
    @Override
    public void onAnimationStart(Animator animation) {
        // 动画开始
    }

    @Override
    public void onAnimationEnd(Animator animation) {
        // 动画结束
    }

    @Override
    public void onAnimationCancel(Animator animation) {}

    @Override
    public void onAnimationRepeat(Animator animation) {}
});
```

### 其它动画

#### 布局切换动画

通过 TransitionManager 处理布局变化的方式，TransitionManager 在工作时，会自动捕获 View 的起始状态（Start State）和结束状态（End State），然后通过类似 ObjectAnimator 的机制，在两者之间创建平滑的过渡。

```java
// 指定动画发生所在的容器
ViewGroup root = findViewById(R.id.main_layout);

// 该方法会让系统会记录下此时容器内所有子View的位置、大小、可见性等信息。
// 默认是AutoTransition（淡入淡出 + 移动 + 缩放）
// 还有Fade 淡入淡出；Slide 滑动进入/退出；Explode 从中心向四周扩散或向中心聚集；ChangeBounds 动态变更大小或未知；当然也能自定义
TransitionManager.beginDelayedTransition(root);

// 直接修改View的状态，在下一帧渲染时，系统会对比修改前后的差异，自动生成平滑的过渡动画
textView.setText("文字变长了，容器会平滑撑开");
imageView.setVisibility(View.GONE); // 这个View会平滑淡出并消失，下方的View会平滑滑上来
```

#### Activity 切换动画

通过指定的 XML 补间动画来实现进入和退出的效果，另外它必须在 startActivity() 或 finish() 调用之后紧接着调用。

```java
Intent intent = new Intent(this, SecondActivity.class);
startActivity(intent);
// 参数1：进入的Activity的动画；参数2：退出的Activity的动画
overridePendingTransition(R.anim.slide_in_right, R.anim.slide_out_left);
```

如果你希望整个 App 的 Activity 切换风格统一（比如全是淡入淡出），可以在 AndroidManifest.xml 的主题里全局配置。

```xml
<style name="AppTheme" parent="Theme.AppCompat.Light">
    <item name="android:windowAnimationStyle">@style/MyActivityAnimation</item>
</style>

<style name="MyActivityAnimation">
    <item name="android:activityOpenEnterAnimation">@anim/fade_in</item>
    <item name="android:activityOpenExitAnimation">@anim/fade_out</item>
    <item name="android:activityCloseEnterAnimation">@anim/fade_in</item>
    <item name="android:activityCloseExitAnimation">@anim/fade_out</item>
</style>
```

#### Fragment 切换动画

用补间动画实现即可。（补间动画不是只是 Canvas 层的变化？Fragment 动画的本质是“先排版，后位移绘图”，在动画还没开始前，新 Fragment 的 View 在系统坐标系里已经处于目标位置）

```java
getSupportFragmentManager().beginTransaction()
    // 参数：进入动画，退出动画，（返回栈）弹入动画，（返回栈）弹出动画
    .setCustomAnimations(
        R.anim.slide_in_right,  // 新Fragment进入
        R.anim.slide_out_left,  // 当前Fragment退出
        R.anim.slide_in_left,   // 按返回键时，上一个Fragment回来
        R.anim.slide_out_right  // 按返回键时，当前Fragment销毁
    )
    .replace(R.id.container, new SecondFragment())
    .addToBackStack(null)
    .commit();
```

#### RecyclerView 的动画

简单说一下。

```java
// 插入 
notifyItemInserted(int position)
// 删除
notifyItemRemoved(int position)
// 更新
notifyItemChanged(int position)
// 移动
notifyItemMoved(int from, int to)
```

```java
DefaultItemAnimator animator = new DefaultItemAnimator();
// 设置动画时长
animator.setAddDuration(500);    // 添加动画时长
animator.setRemoveDuration(500); // 删除动画时长
animator.setMoveDuration(1000);  // 移动动画时长
animator.setChangeDuration(500); // 更新动画时长
recyclerView.setItemAnimator(animator);
```

```java
// 设置新数据时触发进场动画
// 加载布局动画配置文件
Animation animation = AnimationUtils.loadAnimation(context, R.anim.item_slide_in);
// 创建LayoutAnimationController对象
LayoutAnimationController controller = new LayoutAnimationController(animation);
// 设置参数
controller.setDelay(0.15f);
controller.setOrder(LayoutAnimationController.ORDER_NORMAL);
// 应用到RecyclerView
recyclerView.setLayoutAnimation(controller);
```

```java
// 自定义动画，不太会自定义
public class ScaleInItemAnimator extends DefaultItemAnimator {

    // 重写animateAdd方法来自定义添加动画
    @Override
    public boolean animateAdd(final RecyclerView.ViewHolder holder) {
        // 1. 初始化状态：将View缩小并透明
        holder.itemView.setScaleX(0f);
        holder.itemView.setScaleY(0f);
        holder.itemView.setAlpha(0f);

        // 2. 执行动画
        holder.itemView.animate()
                .scaleX(1f)
                .scaleY(1f)
                .alpha(1f)
                .setDuration(getAddDuration())
                .setListener(new AnimatorListenerAdapter() {
                    @Override
                    public void onAnimationStart(Animator animation) {
                        dispatchAddStarting(holder); // 必须调用，通知RecyclerView动画开始
                    }

                    @Override
                    public void onAnimationEnd(Animator animation) {
                        dispatchAddFinished(holder); // 必须调用，通知RecyclerView动画结束，回收状态
                        holder.itemView.animate().setListener(null); // 清除监听器，防泄漏
                    }
                    
                    @Override
                    public void onAnimationCancel(Animator animation) {
                        // 如果动画被取消，需要恢复默认状态
                        holder.itemView.setScaleX(1f);
                        holder.itemView.setScaleY(1f);
                        holder.itemView.setAlpha(1f);
                    }
                }).start();

        return false; // 返回false表示我们自己处理了动画
    }
}

recyclerView.setItemAnimator(new ScaleInItemAnimator());
```

### 动画总结

#### 使用动画的注意事项

+ OOM 问题：加载大量高分辨率图片做帧动画，依然是内存杀手。尽量避免在 UI 上使用长序列的帧动画，如果需要，优先考虑 Lottie（使用 JSON 文件描述动画，性能远超图片帧）。

+ 内存泄露：属性动画持有 View 的引用，View 持有 Context。Animator 这些对象本身不是静态的，但这些动画最终都会注册到全局单例 AnimationHandler 上。所以如果不 cancel，就导致 `AnimationHandler -> Animator -> View -> Activity`，这个引用链一直存在，最终对象无法被 GC，从而内存泄漏。

  另外为什么 AnimationHandler 做成单例呢？这样只需要注册一个监听去处理 Choreographer 信号，然后所有统一计算、统一刷新，更加流畅。

  如果各自刷，动画 A 可能在第 2ms 触发 invalidate，动画 B 在第 10ms 触发，动画 C 在第 14ms 触发。虽然最终合并成一次绘制，但主线程在这一帧里被“骚扰”了三次，每次都要去处理这个逻辑。全用 AnimationHandler 在 Vsync 信号到来的那一刻，统一刷新，只需处理一次。而且也减轻了主线程消息队列 Handler 的压力。

  每个动画如果是自己刷，它获取到的“当前时间”是不准的。动画 A 计算时，系统时间是 1000ms。动画 B 计算时，因为线程调度，时间已经到了 1002ms，最终计算出来的目标位置，可能有微小的撕裂。AnimationHandler 每次刷新时，会从 Choreographer 那里拿一个固定的时间戳，它会把这个同一个时间戳传给所有动画。所有动画都基于“同一个时刻”来计算位置，这样会更好些。

+ 不要使用 px：使用 px 会导致不同密度屏幕下的 UI 错位。始终使用 dp 或相关转换方法。

+ 硬件加速：现在 Android 默认几乎全局开启硬件加速。在处理复杂动画或大量图形操作时，确保没有在 View 层级显式关闭它，这是保证 60fps/120fps 流畅度的前提。

+ View 动画（补间动画）在原位置触发，属性动画在新位置触发。另外现在主流开发全是属性动画，View 动画很少使用。

+ 优先使用 ViewPropertyAnimator，它性能更好，且链式调用非常简洁，自动处理了多属性动画的同步。

#### 各个动画适用的场景

| 动画类型     | 改变实际属性 | 适用对象  | 性能/复杂度 | 首选场景                     |
| ------------ | ------------ | --------- | ----------- | ---------------------------- |
| **补间动画** | 否           | 仅限 View | 极快 / 低   | 简单的 UI 动画（比较少用）   |
| **帧动画**   | 否           | 仅限 View | 较慢 / 低   | 复杂效果                     |
| **属性动画** | 是           | 任意对象  | 快 / 中     | 大部分交互式 UI 动画（首选） |

对于帧动画，可以考虑 Lottie（Airbnb 开源的动画库）进行替换，它通过解析 JSON 文件实现矢量动画，性能和包体积都优于帧动画。

Adobe 的插件 Bodymovin 将动画的每一个关键帧、图层、形状、颜色、路径、变换（平移/缩放/旋转）等信息记录在一个结构化的 JSON 文件中。Lottie 库将这个 JSON 映射成 Java 的对象 LottieComposition，来存储这些数据。然后会将 AE 里的图层对应映射为内部的层级绘制对象，播放时，根据 LottieComposition 存储的这些基础数据，动态根据当前时间戳实时计算关键帧的插值状态（如路径形状、透明度等），然后调用 Android 的 Canvas 和 Paint 接口在屏幕上现场“临摹”出每一帧画面。这样大大减小了帧动画的内存占用，并且因为是矢量动画（不是基于图片的颜色块，而是存储绘图指令），支持各种分辨率。并且也不是很耗 CPU、GPU（取决于图的复杂程度）。

```xml
<com.airbnb.android.lottie.LottieAnimationView
    android:id="@+id/lottieAnimationView"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    app:lottie_fileName="loading.json"
    app:lottie_autoPlay="true"
    app:lottie_loop="true" />
```

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        LottieAnimationView animationView = findViewById(R.id.lottieAnimationView);

        // 1.动态设置动画文件 (可选，如果XML已设置则不需要)
        // animationView.setAnimation("loading.json");

        // 2.控制播放
        animationView.playAnimation();

        // 3.动态修改属性（例如：把动画中的某个颜色改掉）
        // animationView.addValueCallback(...);

        // 4.监听动画状态
        animationView.addAnimatorListener(new Animator.AnimatorListener() {
            @Override
            public void onAnimationStart(Animator animation) {
                // 动画开始
            }

            @Override
            public void onAnimationEnd(Animator animation) {
                // 动画结束
            }

            @Override
            public void onAnimationCancel(Animator animation) {}

            @Override
            public void onAnimationRepeat(Animator animation) {}
        });
    }
}
```

















