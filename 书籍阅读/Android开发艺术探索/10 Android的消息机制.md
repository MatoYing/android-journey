普通线程执行完 run() 方法就结束了，Looper 让线程变成一个**永不退出的循环**，持续接收和处理消息。

## 普通线程

```java
new Thread(() -> {
    doWork();
    // 执行完就结束，线程销毁
}).start();
```


- 生命周期：`run()` 跑完 → 线程结束
- 执行方式：一次性，跑完就没了
- 想再跑？再开一个新线程

## Looper 线程

```java
// Android 主线程伪代码
Looper.loop() {
    while (true) {           // 永远不退出
        Message msg = queue.take();  // 没消息就阻塞等待
        msg.target.dispatchMessage(msg); // 有消息就执行
    }
}
```


- 生命周期：应用不退出，循环就不停
- 执行方式：循环取消息执行，跑完一个接着跑下一个
- 想再跑？往消息队列里丢一条就行，不用开新线程

## 核心区别

|              | 普通线程                 | Looper 线程                 |
| ------------ | ------------------------ | --------------------------- |
| **生命周期** | 执行完即销毁             | 持续运行，直到主动退出      |
| **任务模式** | 一次性的                 | 循环接收，持续处理          |
| **线程复用** | 不复用，每次新建         | 同一线程处理多个任务        |
| **通信方式** | 无法向已结束的线程发消息 | 随时可通过 Handler 投递消息 |

Android 主线程就是一个典型的 Looper 线程——它不会结束，一直在循环取消息（UI 事件、触摸事件、定时器回调……），所以你才能随时 `Handler(Looper.getMainLooper()).post()` 往主线程丢任务。





**每个线程最多 1 个 Looper**，但你可以创建多个带 Looper 的线程。

| 线程                                | Looper 数量          |
| ----------------------------------- | -------------------- |
| 主线程                              | 1 个（系统自动创建） |
| 普通 Thread                         | 0 个（默认没有）     |
| HandlerThread                       | 1 个（自动创建）     |
| 自己给 Thread 调 `Looper.prepare()` | 1 个（手动创建）     |

```java
// 主线程：系统自带 Looper
Looper.getMainLooper(); // 全局唯一

// HandlerThread：自带 Looper 的后台线程
HandlerThread t1 = new HandlerThread("Worker1");
t1.start();
// t1.getLooper() — 这个线程自己的 Looper

HandlerThread t2 = new HandlerThread("Worker2");
t2.start();
// t2.getLooper() — 另一个线程自己的 Looper
```


所以规则是：**一个线程只能有一个 Looper，但应用里可以有多个 Looper 线程**，每个线程各管各的消息队列，互不干扰。





普通线程：跑完就结束，是一次性的。
Looper 线程：跑完一个任务不退出，继续循环等下一个任务，是持久运行的。



不是，虽然行为有点像，但底层机制不同：

|                  | 线程池核心线程                    | Looper 线程                        |
| ---------------- | --------------------------------- | ---------------------------------- |
| **不死**         | 是                                | 是                                 |
| **等待方式**     | 阻塞在 `BlockingQueue.take()`     | 阻塞在 `MessageQueue.next()`       |
| **接收的是什么** | `Runnable`（一段代码）            | `Message`（带数据的消息对象）      |
| **谁投递任务**   | 线程池的 `execute()` / `submit()` | `Handler.sendMessage()` / `post()` |
| **支持定时任务** | 不直接支持                        | 支持（`sendMessageDelayed`）       |
| **所属**         | Java 标准库                       | Android 框架                       |

两者都是"活着等任务来"，但用的是完全不同的等待机制和任务格式。

打个比方：
- **核心线程** = 餐厅服务员站在出菜口等着，厨房喊"菜好了"就去端
- **Looper** = 餐厅服务员盯着叫号屏，屏幕显示桌号就去送菜

都是"等活干"，但等的"信号"不一样。
