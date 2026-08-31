















>  https://developer.android.com/studio/profile/capture-heap-dump?hl=zh-cn



![image-20260831193139156](../../../../../../Library/Application Support/typora-user-images/image-20260831193139156.png)



Allocations

+ 该类在当前内存堆中一共有多少个实例对象。
+ 如果一个不应该有很多实例的类（例如 Activity、Fragment、ViewModel、Repository、UseCase、Adapter、Listener、BroadcastReceiver、ContentObserver 等），其 Allocations 显示为 5，通常意味着发生了内存泄漏（前 4 个页面 `onDestroy()` 销毁后未被 GC 回收）。

Shallow Size

+ 该对象**本身**在 JVM 堆内存中所占用的字节大小，**不包含**它引用的其他对象。
+ 对于常规 Java/Kotlin 对象，这个值通常很小（一般只有十几到几十字节），只包含对象头（Object Header）和自身的基本类型字段（如 `int`、`boolean` 等）以及引用指针本身的体积（4/8 字节）。