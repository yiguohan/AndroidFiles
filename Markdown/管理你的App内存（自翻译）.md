[TOC]

# 管理你的App内存

> 注：本文翻译自Android 开发者官方文档，内容可能会有出入。具体内容以英文官方文档为准 https://developer.android.com/topic/performance/memory.html

随机存储器（RAM，即内存）在任何软件开发环境中都是非常有珍贵的资源，但是它在常常被限制物理内存大小的移动操作系统中更加珍贵。尽管ART和Dalvik虚拟机都会执行日常的垃圾收集（GC），但这不代表你可以无视分配与释放App内存的时机和位置。你仍然需要避免引入内存泄漏(通常由于在静态成员变量中持有对象引用)和在合适的生命周期回调中释放各种引用对象。

这篇文章阐述了如何让你在app中主动减少内存的使用。更多有关于Android操作系统内存管理的信息，请访问[Overview of Android Memory Management](https://developer.android.com/topic/performance/memory-overview.html)。

## 查看可用内存和已用内存

在你解决app内存使用问题之前，你首先需要找到问题所在。Android Studio中的 [Memory Profiler](https://developer.android.com/studio/profile/memory-profiler.html)可以通过以下方式帮助你找到并且判断内存问题：

1. 看看应用程序如何分配内存。Memory Profiler 展示了一张表示内存使用情况、Java对象分配数量和垃圾回收发生的实时图表。
2. 初始化垃圾回收事件(GC Event)并在app运行时记录Java 堆内存的快照。
3. 记录app的内存分配然后煎炒所有已分配的内存，查看每一个分配的堆栈跟踪，跳进Android Stuido 编辑器中对应的代码。

### 释放内存以响应事件

就像在 [Overview of Android Memory Management](https://developer.android.com/topic/performance/memory-overview.html)所描述的，Android 可以通过在app中通过多种不同的方式来回收内存，如果必须要为关键任务释放内存甚至直接将整个app杀死。为了进一步帮助平衡系统内存并且避免系统杀死app进程，你可以在Activity中实现接口`ComponentCallbacks2`。它所提供的`onTrimMemory()`回调方法允许你的app在前台或后台时app监听相关事件，并且随后释放对象来相应app的生命周期或系统事件