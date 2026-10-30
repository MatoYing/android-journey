#### 基础语法

> https://www.runoob.com/kotlin/kotlin-tutorial.html

```kotlin
/**
 * Kotlin 核心笔记 (Java 迁移视角)
 */

// 1. 函数声明
// 默认 public，无返回值写 Unit (等同于 void，可省略)
fun sum(a: Int, b: Int): Int = a + b // 单表达式函数（自动推导返回类型）

fun printSum(a: Int, b: Int): Unit { // Unit 显式写法
    println("Sum of $a and $b is ${a + b}") // 字符串模板：$变量 或 ${表达式}
}

// 2. 变量与空安全
val constant: Int = 10  // 相当于 final Int
var mutable: Int = 20   // 普通变量
var nullable: String? = null // 显式允许 null

// 处理 Null 的三种姿势
val len1 = nullable?.length          // 安全调用：若 null 则返回 null
val len2 = nullable?.length ?: 0     // Elvis 操作符：若 null 则返回 0
// val len3 = nullable!!.length      // 非空断言：若 null 则抛出 NPE (慎用)

// 3. 类与属性
// 主构造函数直接写在类名后，val/var 声明即字段+Getter+Setter
class Person(val name: String, var age: Int) {
    // 初始化块，相当于 Java 的构造代码块
    init { println("Initializing $name") } 
}

// Data Class: 自动生成 equals/hashCode/toString/copy
data class User(val id: Long, val name: String)

// 4. 控制流
// when 取代 switch，它是表达式，可以返回值
val result = when (mutable) {
    1 -> "One"
    in 2..10 -> "2 to 10" // 支持区间
    is Int -> "is Integer" // 类型检查
    else -> "Unknown"      // 必须穷举所有分支
}

// if 也是表达式
val max = if (a > b) a else b // 取代三元运算符 a > b ? a : b

// 5. 集合与循环
val list = listOf("A", "B") // 不可变 List (java.util.Collections.unmodifiableList)
val mutableList = mutableListOf("C") // 可变 List

for (item in list) { println(item) } // 迭代器循环
for (i in 0 until 10) { } // 0-9 (左闭右开)
for (i in 0..10) { }      // 0-10 (闭区间)

// 6. 继承与接口
// 默认所有类都是 final，必须加 open 才能被继承
open class Base {
    open fun v() {} // 方法也必须 open 才能重写
}

class Derived : Base() {
    override fun v() {} // 必须显式写 override
}

interface Clickable {
    fun click()               // 抽象方法
    fun show() = println("!") // 默认实现 (Java 8 default)
}

// 7. 类型转换
// is = instanceof, as = 强制转换, as? = 安全转换（失败返回 null）
val str: String? = obj as? String

// 8. 进阶必备 (Java 没有的)
// Object: 单例模式天生支持
object DatabaseConfig { val url = "jdbc:..." }

// 扩展函数，不修改类源码直接扩展功能，不再需要到处写工具类，经常用的方法，直接加到这个类上
// 扩展函数并没有真的修改原始类的字节码，它本质上是静态导出的工具方法，只是调用起来像成员方法
// 定义扩展函数
fun String.addSmile(): String {
    return this + " :)"
}
// 使用方法
val greeting = "Hello"
println(greeting.addSmile()) // 输出: Hello :)
```

#### Flow

协程：

与其让操作系统调度，不如我自己来，这就是协程。

说白了就是系统调度需要走流程（这里申请一下，那里盖个章），自己办事不求人自然快的多

好处就是减少了用户态和核心态的切换

IO多路复用是除进程模型线程模型之外，实现并发的一种方式，一个进程调用select函数，同时等待多个事件，事件到达后才获得控制权开始执行该对应的代码，事件到达前处于挂起状态，因此又叫事件驱动程序。缺点是编码复杂，要把所有的事件逻辑写在一个程序里；又因为只有一个线程，无法利用多cpu。Node.js、nginx等高性能服务器使用的都是IO多路复用的事件编程方式，主要因为其比起进程线程具有明显的性能优势。Go基于多个channel（事件）的select也是一种IO多路复用。

> https://www.bilibili.com/video/BV1QD421J7oD









**RxJava VS. Flow 实现**







```kotlin
data class DriveCalibrationConfig() 用法
```















https://blog.csdn.net/guolin_blog/article/details/109787732

https://blog.csdn.net/weixin_37794278/article/details/149777742





















