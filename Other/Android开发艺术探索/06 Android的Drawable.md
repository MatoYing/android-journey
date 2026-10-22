#### 基础介绍

Drawable 和 View 有什么差别呢？

Drawable 主要解决的就是图形绘制相关的问题，主要就是利用 Canvas 进行相关操作（它的核心）。而 View 呢，主要解决的是事件分发和布局测量方面的工作，View 承载 Drawable，Drawable 通常作为 View 的背景或内容。

Drawable 是一个抽象基类，所有具体的 Drawable 都继承于它。

```
Drawable (抽象类)
    ├── BitmapDrawable
    ├── GradientDrawable
    ├── VectorDrawable
    ├── NinePatchDrawable
    ├── LayerDrawable
    │    └── TransitionDrawable
    ├── DrawableContainer（状态切换容器——内部类管理）
    │    ├── StateListDrawable (对应 <selector>)
    │    └── LevelListDrawable (等级切换)
    └── DrawableWrapper（装饰器模式）
         ├── InsetDrawable
         ├── ScaleDrawable
         └── ClipDrawable
```

另外 Drawable 的一个核心方法就是 `public abstract void draw(@NonNull Canvas canvas)`，也应证了上面所说的主要利用 Canvas 完成绘制。

#### StateListDrawable（重要）

根据 View 的状态从上到下匹配，匹配到第一个符合条件的条目后就停止。默认状态（没有写状态的项）放在最后。

```xml
<Button
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="Click Me"
    android:background="@drawable/my_button_bg" />
```

```xml
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- 每个<item>定义一种状态下的颜色，系统从上到下匹配，第一个满足所有state条件的item生效。因此要把更具体的状态放前面，默认状态放最后 -->
    <item android:drawable="@drawable/ic_btn_pressed" android:state_pressed="true" />
    <item android:drawable="@drawable/ic_btn_normal" />
</selector>


<!-- 更全面的 -->
<selector xmlns:android="http://schemas.android.com/apk/res/android" >
    <item
        android:drawable="@drawable/ic_btn_pressed"
        android:state_pressed="true"         <!-- 是否被按下，外部控制 -->
        android:state_focused="true"         <!-- 是否获得焦点，外部控制 -->
        android:state_selected="true"        <!-- 是否被选中 -->
        android:state_checkable="true"       <!-- 是否可勾选 -->
        android:state_checked="true"         <!-- 是否已勾选 -->
        android:state_enabled="true"         <!-- 是否可用（启用/禁用） -->
        android:state_window_focused="true"  <!-- 所在窗口是否拥有焦点，外部控制 -->
    />
</selector>
```

#### ~~ShapeDrawable~~ GradientDrawable（重要）

定义几何参数，在渲染阶段通过绘图指令动态生成的矢量图形资源。

因为 xml 的标签是 `<shape>`，大家普遍都认为这个是 ShapeDrawable，但 `<shape>` 其实是 GradientDrawable。ShapeDrawable 其实是更底层的类，是面向 Java 的，纯代码去绘制几何图形，

```xml
<shape xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="rectangle"> <!-- 形状：rectangle, oval, line, ring -->

    <!-- 填充颜色 -->
    <solid android:color="#4CAF50" />

    <!-- 描边/边框 -->
    <stroke
        android:width="2dp"
        android:color="#2E7D32"
        android:dashGap="4dp"   <!-- 虚线间距 -->
        android:dashWidth="8dp" /> <!-- 虚线宽度 -->

    <!-- 圆角 -->
    <corners
        android:radius="10dp"
        android:topLeftRadius="20dp" /> <!-- 也可以单独指定某个角 -->

    <!-- 渐变色 (与 solid 互斥) -->
    <gradient
        android:startColor="#4CAF50"
        android:endColor="#81C784"
        android:angle="45"
        android:type="linear" />

    <!-- 内边距 -->
    <padding
        android:bottom="8dp"
        android:left="8dp"
        android:right="8dp"
        android:top="8dp" />
</shape>
```

#### LayerDrawable（重要）

像 Photoshop 的图层一样，一层一层往上叠。后写的 `<item>` 叠在先写的上面。

```xml
<layer-list xmlns:android="http://schemas.android.com/apk/res/android">
    
    <!-- 第一层：阴影层 (偏移 2dp) -->
    <item
        android:left="2dp"
        android:top="2dp">
        <shape>
            <solid android:color="#33000000" /> <!-- 半透明黑 -->
            <corners android:radius="5dp" />
        </shape>
    </item>

    <!-- 第二层：主体层 (稍微往上/左偏，露出底层的阴影) -->
    <item
        android:bottom="2dp"
        android:right="2dp">
        <shape>
            <solid android:color="#FFFFFF" />
            <stroke android:width="1dp" android:color="#000000" />
            <corners android:radius="5dp" />
        </shape>
    </item>
</layer-list>
```

#### VectorDrawable（重要）

首先只要不是由像素点组成的（像 PNG、JPG 那样），而是通过数学公式或描述信息定义的图形，都是**矢量图**。那 VectorDrawable 和 GradientDrawable 有什么差别呢？

首先，因为它们都是矢量图，它们的一个共同点是都能做到完美缩放，不失真。

GradientDrawable，它是“几何描述”，定义简单几何图形的集合。底层原理是对 Canvas 基础几何 API 的直接封装，只能绘制矩形、椭圆、线、环四种固定形状，可以高效处理圆角、边框及渐变。它将 XML 属性直接映射为 Canvas.drawRect、drawRoundRect 或 drawOval 等指令的参数。由于形状是固定的数学模板，系统在绘制时只需传递圆角半径、边框宽度等极少量参数给 GPU，不涉及复杂的路径运算，性能开销小。

VectorDrawable，它是“路径描述”，是复杂路径的集合。它通过一连串坐标指令（Path Data，如 M 移动到、L 画线、C 贝塞尔曲线）勾勒出任意复杂的形状，本质上是路径轨迹的集合。系统解析 pathData 字符串并生成底层的 Path 对象，渲染时最终调用 Canvas.drawPath。这个过程需要将数学曲线在特定分辨率下转换为像素点（这个过程叫光栅化），涉及昂贵的“路径扫描”计算。如果路径包含数千个坐标点，CPU 解析路径和 GPU 执行光栅化的压力会剧增，耗时会显著增加。（GradientDrawable 虽然最终也需光栅化，但它利用的是 GPU 硬件加速的“绘图原语”，系统只需通过极少量的数学参数，如半径、坐标，即可快速填充像素，不涉及复杂的路径解析与逐点扫描）。它通常由 UI 设计师提供的 SVG 经 Android Studio 的 Vector Asset 工具转换而成。

```xml
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"      <!-- 逻辑上的宽度 -->
    android:height="24dp"     <!-- 逻辑上的高度 -->
    android:viewportWidth="24"  <!-- 画布视口宽度 -->
    android:viewportHeight="24"> <!-- 画布视口高度 -->

    <!-- pathData 就是绘图指令，比如 M=移动, L=画线, Z=闭合 -->
    <path
        android:fillColor="#FF000000"
        android:pathData="M20,11H7.83l5.59,-5.59L12,4l-8,8 8,8 1.41,-1.41L7.83,13H20v-2z" />
        
</vector>
```

#### InsetDrawable

根据 Google 的设计规范，按钮点击热区建议不小于 48dp。但 UI 设计稿经常给一个很小的图标，比如 24dp，想把它热曲变大，首先看一个利用 `android:padding` 的错误用法。

```xml
<!-- 错误用法 -->
<ImageButton
    android:layout_width="48dp"
    android:layout_height="48dp"
    android:padding="12dp"  <!-- 同时影响src和background -->
    android:scaleType="centerInside" <!-- 只控制src，不控制background，这个表示保持图片完整的进行缩放，centerCrop为了填满整个框可能会裁剪 -->
    android:src="@drawable/ic_back_arrow"  <!-- 前景内容，保持图片原有比例，可配合scaleType控制缩放 -->
    android:background="@drawable/bg_circle_red" /> <!-- 背景，会拉伸填满整个View大小，不参与scaleType控制 -->
```

首先说明一下 src 和 background 的区别

| **特性**     | android:background                | android:src                          |
| ------------ | --------------------------------- | ------------------------------------ |
| **适用范围** | 所有 View                         | 主要是 ImageView                     |
| **图片比例** | 强制填满 View，容易导致拉伸变形。 | 可通过 `scaleType` 保持比例。        |
| **层级**     | 最先绘制（最底层）。              | 在背景之后绘制（盖在背景上）。       |
| **主要用途** | 装饰控件边缘、底色、阴影。        | 展示核心数据内容（如头像、商品图）。 |

像上面的写法，src 会因为 padding 进行缩放；background 会被强制拉升为 48dp。正确的写法应该是下面这样，src 和 background 都不会被影响，并且按钮点击热区变大了：

```xml
<inset xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/bg_circle_red"
    android:insetLeft="12dp"
    android:insetRight="12dp"
    android:insetTop="12dp"
    android:insetBottom="12dp">
</inset>

<ImageButton
    android:layout_width="48dp"
    android:layout_height="48dp"
    android:scaleType="centerInside"
    android:src="@drawable/ic_back_arrow"
    android:background="@drawable/bg_circle_inset" />  <!-- 用InsetDrawable代替 -->
```

当然也能通过组装一个 Layout 来实现，然后通过控制内边距这些实现，但这会增加了一个 View 层级，影响性能。

还有些只能用 Drawable 的地方，比如 RecyclerView 的分割线，你想要一个分割线，但分割线左右需要留出 20dp 的空白。我们会使用 DividerItemDecoration 来设置分割线。这个类只接受一个 Drawable 对象，它不接受 Layout 布局，这就需要用到 InsetDrawable 的方式。（当然也可以让 UI 直接给个大小合适的）

```xml
<inset xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/line_shape"
    android:insetLeft="20dp"
    android:insetRight="20dp">
</inset>
```

#### ClipDrawable

根据 Level 的百分比，来裁剪另一个 Drawable。Level 的取值范围是 0（完全隐藏）到 10000（完全显示）。一般用来做进度条这种。

```xml
<clip xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/image_to_clip"
    android:clipOrientation="horizontal"
    android:gravity="left" /> <!-- left 表示左边固定，从右边开始裁剪 -->
```

它必须配合代码才能动起来

```java
ImageView img = findViewById(R.id.my_image);
ClipDrawable drawable = (ClipDrawable) img.getDrawable();
// 设置为5000，表示显示图片左半部分的50%
drawable.setLevel(5000);
```

#### BitmapDrawable

它本质上就是对一张像素图片（PNG、JPG）的包装，虽然平时我们直接引用 `@drawable/xxx_image`，但通过 BitmapDrawable 这种方式，可以有更多对图片的高级操作。

```xml
<bitmap xmlns:android="http://schemas.android.com/apk/res/android"
    android:src="@drawable/your_real_image"  <!-- 原始图片资源 -->
    android:gravity="center"                 <!-- 当 View 大于图片时，图片居中 -->
    android:antialias="true"                 <!-- 开启抗锯齿，旋转或缩放时边缘更平滑 -->
    android:dither="true"                    <!-- 开启抖动，在低色深屏幕上防止色带 -->
    android:filter="true"                    <!-- 开启过滤，缩放时保持平滑 -->
    android:tileMode="repeat" />              <!-- 平铺模式 -->
```

tileMode：

+ disabled (默认)：不平铺，拉伸占满整个 View。
+ clamp：拉伸边缘像素（空白区域用图片最边缘那一排像素的颜色填充）
+ repeat：平铺（当原始图片的尺寸小于显示它的 View（视图）时，系统会像铺瓷砖一样，把这张图片重复、拼贴地绘制，从而填满整个背景区域）
+ mirror：镜像平铺（第一块是正的，第二块是反的，以此类推）。

antialias：因为所有图像都是由一个个很小的方形像素组成的，当你画一条完全水平或完全垂直的线时，边缘是整齐的；但是但当你画一条斜线或圆形时，你就会在边缘看到，一段一段的“楼梯”形状，这些“楼梯”就是锯齿。开启抗锯齿后，在锯齿边缘的过渡区域，会添加半透明的过渡像素，比如黑色像素和白色像素间会添加灰色像素（离黑色近的深灰，离白色近的浅灰），这样在视觉上会更平滑（分辨率越高，效果越好）。

dither：适用比较老的手机，支持的颜色比较少。抖动会根据算法在颜色交界处交错排列不同色的像素点，避免出现阶梯状线条（色带）

#### LevelListDrawable（不重要）

它像是一个 switch 语句。它管理一组图片，每张图片对应一个“数字范围”。你给它一个数字，它就显示对应的图。

```xml
<level-list xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:drawable="@drawable/ic_battery_empty" android:maxLevel="0" />
    <item android:drawable="@drawable/ic_battery_low" android:maxLevel="20" />
    <item android:drawable="@drawable/ic_battery_full" android:maxLevel="100" />
</level-list>
```

```java
// 设为 15，系统会自动匹配到 maxLevel="20" 的那张图
imageView.setImageLevel(15);
```

#### TransitionDrawable（不重要）

它是 LayerDrawable 的子类，但只包含两层。它的唯一作用是：让第一层和第二层之间实现平滑的“淡入淡出”切换效果。比如一个灯泡图片，从“熄灭状态”平滑过渡到“点亮状态”，而不是瞬间闪过去。现在不太适用了，完全可以直接使用属性动画实现。

```xml
<transition xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:drawable="@drawable/light_off" />
    <item android:drawable="@drawable/light_on" />
</transition>
```

```java
TransitionDrawable td = (TransitionDrawable) view.getBackground();
td.startTransition(500); // 在0.5秒内从off渐变到on
```

 #### ScaleDrawable（不重要）

它包装另一个 Drawable，并根据当前的 level（等级）来决定这个图片的显示比例。几乎不使用。

```xml
<scale xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/my_image"
    android:scaleGravity="center"
    android:scaleHeight="100%"  <!-- 高度随 level 变化 -->
    android:scaleWidth="100%" /> <!-- 宽度随 level 变化 -->
```

```java
Drawable sd = view.getBackground();
sd.setLevel(5000); // 显示原图 50% 的大小
sd.setLevel(10000); // 100% 完全显示
```

#### NinePatchDrawable（不重要）

TODO









#### 自定义 Drawable

如果要绘制非标准的复杂几何图形、或者高度自定义来实现某些需求时，可以用到自定义 Drawable。

```java
public class CircleDrawable extends Drawable {
    private final Paint mPaint;
    private int mRadius;

    public CircleDrawable(int color) {
        mPaint = new Paint(Paint.ANTI_ALIAS_FLAG); // 抗锯齿
        mPaint.setColor(color);
    }

    // 最核心的方法。在这里使用Canvas和Paint绘制想要的图形
    @Override
    public void draw(Canvas canvas) {
        // 获取当前Drawable的绘制边界（即View给它分配的大小）
        Rect bounds = getBounds();
        
        // 计算中心点坐标
        float cx = bounds.exactCenterX();
        float cy = bounds.exactCenterY();
        
        // 计算半径（取宽高最小的一半）
        mRadius = Math.min(bounds.width(), bounds.height()) / 2;

        // 绘制圆形
        canvas.drawCircle(cx, cy, mRadius, mPaint);
    }

    // 设置透明度（0-255）
    @Override
    public void setAlpha(int alpha) {
        mPaint.setAlpha(alpha);
        invalidateSelf(); // 重新绘制
    }

    // 设置颜色滤镜
    @Override
    public void setColorFilter(ColorFilter colorFilter) {
        mPaint.setColorFilter(colorFilter);
        invalidateSelf();
    }

    // 返回透明度格式
    @Override
    public int getOpacity() {
        // 告诉系统这个图形是透明、不透明还是半透明的
        return PixelFormat.TRANSLUCENT;
    }
  
    // 当View的大小改变时，系统会调用onBoundsChange
    @Override
    protected void onBoundsChange(Rect bounds) {
        super.onBoundsChange(bounds);
        // 在这里根据新边界更新内部参数
    }
  
    // 支持状态切换 
    @Override
    public boolean isStateful() {
        return true; // 声明支持状态改变
    }
    // 实现Drawable在“按下”或“禁用”时显示不同效果
    @Override
    protected boolean onStateChange(int[] state) {
        // 遍历state数组，判断是否有android.R.attr.state_pressed 
        // 如果改变了颜色等属性，返回true触发重绘
        invalidateSelf();
        return true;
    }
  
    // 有些Drawable（如图片）有固定的原始尺寸。通过重写这两个方法告诉View它的默认大小；有些时候不用，比如用作View的背景时，Drawable会被拉伸至View的同等大小
    @Override
    public int getIntrinsicWidth() {
        return dpToPx(100);  // 默认宽度100dp
    }
    @Override
    public int getIntrinsicHeight() {
        return dpToPx(100);  // 默认高度100dp
    }
}
```

```java
ImageView imageView = findViewById(R.id.my_image_view);
CircleDrawable circleDrawable = new CircleDrawable(Color.RED);

// 设置为 ImageView 的内容
imageView.setImageDrawable(circleDrawable);

// 或者设置为 View 的背景
// view.setBackground(circleDrawable);
```

