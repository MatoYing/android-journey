```
Profiler/
├── 00_概览与入口.md                    # Profiler 是什么、怎么打开、工具全景
│
├── CPU/
│   ├── 01_CPU_Profiler_概览.md         # CPU 分析的三种方式对比与选型
│   ├── 02_System_Trace.md             # 系统级追踪（调度、I/O、内核事件）
│   ├── 03_Callstack_Sample.md         # 调用栈采样找热点
│   └── 04_Method_Recording.md         # Java/Kotlin 方法级录制
│
├── Memory/
│   ├── 05_Memory_Profiler_概览.md      # 内存分析的三种方式对比与选型
│   ├── 06_Heap_Dump.md                # 堆转储分析（含内存泄漏排查步骤）
│   ├── 07_Java_Kotlin_Allocations.md  # Java/Kotlin 实时分配追踪
│   └── 08_Native_Allocations.md       # Native (C/C++) 内存追踪
│
└── Telemetry/
    └── 09_Live_Telemetry.md            # 实时遥测概览（CPU/内存/帧率）
```


线程：
+ 绿色：线程处于活跃状态或就绪状态——即正在运行或等待 CPU 调度。
+ 黄色：线程处于活跃状态，但正在等待 I/O 操作（如磁盘读写或网络请求）完成后才能继续执行。
+ 灰色：线程处于休眠状态，不消耗 CPU 时间。通常是因为线程所需的资源尚不可用——可能是线程主动进入休眠，也可能是内核将其挂起，直到资源就绪。