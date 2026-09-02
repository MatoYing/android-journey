---
typora-copy-images-to: ../../../assets
---











#### Heap Dump（.hprof）

Heap Dump 是 Java/Android 虚拟机在某一时刻把整个堆内存（Heap）中的对象状态完整序列化到磁盘的文件，文件扩展名为 .hprof（Heap Profile 的缩写）。简单来说就是堆内存的瞬时快照，已被 GC 回收的对象不会存在于快照中。



如何采集：



```
关键步骤解释：
先 GC：清掉不可达的死对象，避免干扰
操作场景：比如反复进出某个页面
再 GC：如果对象真的泄漏，GC 也回收不掉，它还会留在堆里
Dump：把此刻的堆状态冻结下来
分析：在 Profiler 里按 Allocations、Retained Size 等指标排序，找到可疑对象，顺着引用链追到 GC Root，找出谁在不合理地持有它
```



```
在 Android 开发中有几种方式：
Android Studio Profiler → Memory 面板点击 "Capture heap dump" 按钮
命令行 → adb shell am dumpheap <pid> /data/local/tmp/dump.hprof，然后 adb pull 拉出来
代码中触发 → Debug.dumpHprofData(path) （需要 debug 包）
```



```
核心用途
排查内存问题，主要包括：
内存泄漏：Activity/Fragment 退出后仍被持有，GC 回收不了
内存占用过高：找出哪些对象吃掉了大量内存
大对象定位：发现异常大的 Bitmap、List、缓存等
对象数量异常：某类实例数量远超预期（如单例变成了多实例）
```



```
注意事项
hprof 文件可能很大（几十 MB 到几百 MB），取决于应用堆大小
dump 过程会 暂停应用（STW, Stop-The-World），不要在 release 包或生产环境使用
Android 的 hprof 格式和标准 JVM 的略有差异，Android Studio 会自动转换；如果用 MAT（Memory Analyzer Tool）分析，需要先用 hprof-conv 工具转换格式
```



如果是独立的单例/缓存泄漏，需要开发者结合 **`Allocations` 数量异常膨胀** 或 **`Retained Size` 过大** 自行排查。



>  https://developer.android.com/studio/profile/capture-heap-dump?hl=zh-cn

![image-20260901152654224](../../../assets/image-20260901152654224.png)

Allocations

+ 表示当前堆中**该类**的存活**实例数量**。
+ 如果一个不应该有很多实例的类，比如一个 Activity 的 Allocations 显示为 5，看到 Activity 数量 > 1 几乎可以断定有泄漏（前 4 个页面经历 `onDestroy()` 销毁后未被 GC 回收），有引用链在阻止它们被回收。

Shallow Size

+ 该类的对象实例本身在 JVM 堆内存中所占用的字节大小，不包含它引用的其他对象。
+ 对于常规 Java/Kotlin 对象，这个值通常很小（也有很大的，比如 RecyclerView，继承链很深，东西很多），只包含对象头和自身的基本类型字段（如 int、boolean 等）以及引用指针本身的体积。例如一个只含两个 int 字段的对象，Shallow Size 大约是 16（对象头）+ 8（两个 int）= 24 字节左右。
+ 但它并不反映对象的真实内存状况，因为大部分对象都会引用其它对象。
+ 同一个类的所有实例，其 Shallow Size 是一模一样的。但对于数组，它的 Shallow Size 取决于数组长度。但普通对象（如List、User 对象）的 Shallow Size 通常只有 24 - 40 字节（List 里面都是引用，所以不会影响 Shallow Size；但引用特别的多的话，Retained Size 可能会很大）。

Native Size

+ 表示该类的对象实例在 Native 堆（C/C++ 层） 中占用的内存字节数。
+ Java 对象本身在 JVM 堆中占的内存不大，但部分特定类型的对象会在底层通过 Native 代码分配额外的内存，且这部分内存可能远大于对象本身的 Shallow Size。
+ 日常开发中，95% 以上的对象都没有 Native 内存开销，可能会遇到的如下：
  + Bitmap：Java 层仅仅是一个轻量级的 Bitmap 外壳（Shallow Size 很小，可能只有几十字节），真实的像素矩阵会存放在 Native 内存中，会达到 MB 级。
  + Paint / Canvas / Path：绘图相关，底层都是 Skia 图形引擎的 C++ 对象，每个都有对应的 Native 内存。
  + MediaPlayer / ExoPlayer：音视频解码缓冲区、解码器实例都在 Native 层。
  + DirectByteBuffer：NIO。
  + Surface / SurfaceView / TextureView：图形缓冲区（GraphicBuffer）分配在 Native 层。
  + SQLiteDatabase / Cursor：SQLite 引擎是纯 C 实现，查询结果缓存在 Native 内存。
  + WebView：整个浏览器内核（Chromium）在 Native 层，一个 WebView 可能占 几十 MB 的 Native 内存。

Retained Size

+ 表示如果彻底回收/销毁这个类所有对象实例，系统总共能释放出多少字节内存。
+ Retained Size = 自身大小（Shallow Size + Native Size） + 该对象独占的所有对象的 Shallow Size、Native Size 之和（“独占引用”，如果某个子对象只有这一条引用链能到达，那它算独占；如果有其它路径也能到达该子对象，则不算）。
+ **这是判断内存影响最重要的指标。**当发现内存占用高时，永远先看 Retained Size 的倒序排列，解决 Retained Size 最大的那个对象，通常能产生“立竿见影”的效果。比如一个 Activity 实例的 Retained Size 是 20MB，说明如果它被泄漏，就会白白占住 20MB 内存。总之 Retained Size 越大，泄漏危害会越大。

![image-20260901152820236](../../../assets/image-20260901152820236.png)

![image-20260901152912355](../../../assets/image-20260901152912355.png)

+ app heap：对于 APP 开发者唯一需要关注的。你的应用在运行期间，自己通过代码 new 出来、业务逻辑所产生的所有对象。下面呢俩是给 Android 系统底层工程师、Framework 定制开发人员（如车载车机/手机厂商 ROM 团队）
+ image heap：系统编译期提前固化好的只读类与核心对象镜像，随开机直接映射到内存，所有进程共享且 GC 永远不会回收。
+ zygote heap：系统开机时 Zygote 进程预加载的框架层资源，App 启动时直接通过写时复制（COW）机制继承并默认共享。

![image-20260901163222276](../../../assets/image-20260901163222276.png)

+ Show all classes：展示当前堆内存中所有的类，不做任何过滤。你的业务代码、第三方开源库、系统 Framework 类、JDK 基础类（`java.*`）等全部在列。
+ Show activity/fragment leaks：仅显示泄漏的 Activity/Fragment。Profiler 会自动运行检测算法，找出生命周期已经执行了 `onDestroy()`（即页面已销毁），但在堆中依然被强引用无法被 GC 回收的 Activity 或 Fragment 实例。只列出被系统判定为已经泄漏的 UI 组件类（带黄色感叹号警告）。
+ Show project classes：仅显示当前工程的类，自动过滤掉系统的 SDK 类（`android.*`、`java.*`）和第三方依赖库，只留下你自己的包名下的类。

![image-20260901183053177](../../../assets/image-20260901183053177.png)

这个上方是“类”，点击后，下面会显示出该类的所有“实例”。

+ Depth：GC 引用深度，从系统中最顶级的“根节点（GC Root，例如栈变量、静态变量、JNI 全局引用）”出发，经过几层引用指针才能找到当前这个对象。

  ```
  GC Root（线程栈）
    └── Activity           ← Depth 1
          └── ViewGroup    ← Depth 2
                └── View   ← Depth 3
                      └── Listener ← Depth 4
  ```

+ Native Size、Shallow Size、Retained Size：同上，这个是单实例的大小

+ Fields：（向下看）表示这个类有哪些字段，以及每个字段的值。

+ References：（向上看）展示在内存中谁持有了当前对象的引用，并一路向上追溯到系统最顶级的根节点（GC Root）。

+ Show nearest GC root only：勾选代表只显示到最近 GC Root 的最短路径；不勾选代表显示所有引用这个对象的路径，一个对象在复杂的 Android 系统中可能被几十个地方间接弱引用或引用，如果不勾选，References 树会极其庞大繁杂。



虽然它**绝不会漏掉“这个对象发生泄漏”的事实**，但在实际定位原因时，它展示出来的引用链**不一定是你代码里写出的那个错误点**。







**检测重复的 BitMap**

目前 Profiler 的重复检测功能只针对 Bitmap。因为 Bitmap 是 Android 中最容易出现重复加载的对象，且单个实例的 Native 内存占用大，重复时的浪费也最为显著。

![image-20260902005835944](../../../assets/image-20260902005835944.png)

**查找内存泄漏**

Profiler 检测内存泄漏的原理

+ 遍历堆中所有对象，筛选出 Activity 和 Fragment 的子类实例。
+ 通过检查内部标志位，判断哪些已经被"销毁"。Activity，检查 mDestroyed 字段是否为 true；Fragment，检查 mLifecycleRegistry 的当前状态是否为 DESTROYED。如果已标记销毁，说明 `onDestroy()` 已经执行过了，按理说应该被 GC 回收。
+ 既然 `onDestroy()` 已经执行了，这个对象就应该被 GC 回收。如果它仍然出现在 Heap Dump 里，说明有引用链在阻止它被回收，就会判定为泄漏。

但是像 Service、BroadcastReceiver 、Overlay 等泄漏，Profiler 是检测不出来的。因为这些没有明确的生命周期结束时间。

**内存泄漏的危害**

















#### Java/Kotlin Allocations

在 Memory Profiler 里点 Record 开始录制，它会在一段时间内追踪每一次 new 操作。




是的，在 Android Studio **Profiler** 中完全可以看到这两个指标，但它们在界面上的**列名（Labels）**可能和你用的术语稍有不同。

当你使用 **"Record Java/Kotlin allocations"**（记录内存分配）功能，并选择一段特定的时间范围（Timeline Range）后，下方的表格会列出以下关键信息：

1. Total Allocations (在 Profiler 中显示为 **Allocations**)

- **对应列名：** Allocations
- **含义：** 表示在你选定的这一段时间内，该类**总共被创建（实例化）了多少次**。
- **分析意义：** 它是累计值。如果这个数值很大，即使当前存活的对象很少，也说明代码中存在**“内存抖动”**（Memory Churn）。比如你在 onDraw 或循环里频繁创建对象，会导致这个值飞涨，从而触发频繁的 GC，造成界面卡顿。

2. Live Allocations (在 Profiler 中显示为 **Total Count**)

- **对应列名：** Total Count
- **含义：** 表示在当前选定时间段的**终点时刻**，该类有多少个实例**依然存活在堆中**。
- **计算公式：** Total Count = 记录前的存活数 + 新分配(Allocations) - 已回收(Deallocations)。
- **分析意义：** 这是判断**内存泄漏**的核心指标。如果你反复操作某个页面（例如进出 Activity 5 次），发现该 Activity 的 Total Count 变成了 5，那就说明发生了泄漏。



在哪里查看？

在 Android Studio Profiler 的 **Memory** 面板中，根据你的操作模式，看到的内容会有细微差别：

1. **录制模式 (Record Java/Kotlin Allocations)：**你可以同时看到 **Allocations** (总分配数)、**Deallocations** (回收数) 和 **Total Count** (当前存活数)。这是观察对象“生命周期动态”的最佳方式。
2. **堆转储模式 (Capture Heap Dump)：**主要显示的是 **Allocations** 或 **Instance Count**。因为 Heap Dump 是一个“瞬间的快照”，所以此时看到的 Allocations 实际上指的就是**当前存活的实例数量**（即 Live Allocations）。

总结对照表：

| 你的术语              | Profiler 对应列名 | 关注点             | 理想状态                       |
| --------------------- | ----------------- | ------------------ | ------------------------------ |
| **Total Allocations** | **Allocations**   | 内存抖动、GC 压力  | 数值平稳，不应随短时间操作剧增 |
| **Live Allocations**  | **Total Count**   | 内存泄漏、内存占用 | 操作结束后，该值应回到初始状态 |

分析**内存泄漏**时，重点看 **Total Count**（存活数）；
分析**运行卡顿**时，重点看 **Allocations**（总分配频率）。







#### Native Allocations