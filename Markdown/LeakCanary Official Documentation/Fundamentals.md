[TOC]

# 基础

## 内存泄漏是什么？

在Java运行时环境中，内存泄漏是一种由应用程序保留一个不再需要的对象引用而产生的程序设计错误。由此产生的结果是分配给对象的内存无法回收，最终导致`OutOfMemoryError`崩溃。

举个例子，Android 的Activity实例在`onDestory()`方法调用之后不再需要，而在一个静态字段中保留Activity的引用可能会阻止该实例被GC回收。

## 一般情况下导致内存泄漏的原因

大多数的内存泄漏的原因都归于对象的生命周期相关的bug。以下是一些常见的Android 错误：

- 把Acitivty的上下文储存在对象的一个字段中，从而导致Activity的配置变化不会发生。
- 在生命周期中注册监听器，广播接收器，或者RxJava中的订阅，但是在生命周期结束的时候忘记取消订阅。
- 在静态字段中保存View，并且在View detach的时候忘记清空该字段。

## 为什么我要用LeakCanary?

内存泄漏在Android 应用中是非常常见的。OOM(OutOfMemoryError)是Play Store中的App里最常见的崩溃类型，但通常不会被计算正确。当内存不足时，可以从代码中的任何位置抛出OOM，这意味着每个OOM都有不同的堆栈跟踪，并且它们被视为不同的崩溃。

当我们首次在Square Point Of Sale应用程序中启用LeakCanary时，我们能够找到并修复多个泄漏并将OutOfMemoryError崩溃率降低94％。

## LeakCanary是如何工作的？

### 检测保留的实例

LeakCanary的基础是一个名为LeakSentry的库。LeakSentry挂钩到Android生命周期，自动检测那些已经被销毁并且应该被GC的Activity和Fragment。这些被销毁的实例被传递给`RefWatcher`，并包含对它们的弱引用。您还可以观看不再需要的任何实例，例如 一个detached的View，一个被销毁的Presenter等。

如果在等待5秒并且运行垃圾收集器后未清除弱引用，则被监视的实例被视为*保留*，并且可能泄漏。

### 堆转储

当保留的实例数达到阈值时，LeakCanary将Java堆转储到存储在Android文件系统上的.hprof文件中。 默认阈值为5个保留实例此时应用程序可见，1时则应用程序不可见。

### 分析堆

LeakCanary解析.hprof文件并找到防止保留实例被垃圾收集的引用链：泄漏跟踪。泄漏跟踪也就是从垃圾收集根到保留实例的最短强引用路径的另一种说法。一旦确定了泄漏跟踪，LeakCanary就会利用其内置的Android框架知识来推断泄漏跟踪中的哪些实例正在泄漏。

### 将泄漏分组

使用泄漏状态信息，LeakCanary将参考链缩小到可能泄漏原因的子链，并显示结果。 具有相同因果链的泄漏被认为是相同的泄漏，因此泄漏由相同的子链分组。

## 我该如何修复内存泄漏？

对于每个泄漏实例，LeakCanary计算泄漏跟踪并在其UI中显示：

![leak trace](https://square.github.io/leakcanary/images/leaktrace.png)

泄漏跟踪也记录到Logcat：

```
    ┬
    ├─ leakcanary.internal.InternalLeakCanary
    │    Leaking: NO (it's a GC root and a class is never leaking)
    │    ↓ static InternalLeakCanary.application
    ├─ com.example.leakcanary.ExampleApplication
    │    Leaking: NO (Application is a singleton)
    │    ↓ ExampleApplication.leakedViews
    │                         ~~~~~~~~~~~
    ├─ java.util.ArrayList
    │    Leaking: UNKNOWN
    │    ↓ ArrayList.elementData
    │                ~~~~~~~~~~~
    ├─ java.lang.Object[]
    │    Leaking: UNKNOWN
    │    ↓ array Object[].[0]
    │                     ~~~
    ├─ android.widget.TextView
    │    Leaking: YES (View detached and has parent)
    │    View#mAttachInfo is null (view detached)
    │    View#mParent is set
    │    View.mWindowAttachCount=1
    │    ↓ TextView.mContext
    ╰→ com.example.leakcanary.MainActivity
    ​     Leaking: YES (RefWatcher was watching this and MainActivity#mDestroyed
is true)
```

