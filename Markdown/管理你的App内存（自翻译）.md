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

就像在 [Overview of Android Memory Management](https://developer.android.com/topic/performance/memory-overview.html)所描述的，Android 可以通过在app中通过多种不同的方式来回收内存，如果必须要为关键任务释放内存甚至直接将整个app杀死。为了进一步帮助平衡系统内存并且避免系统杀死app进程，你可以在Activity中实现接口`ComponentCallbacks2`。它所提供的`onTrimMemory()`回调方法允许你的app在前台或后台时app监听相关事件，并且随后释放对象来响应那些预示着系统需要回收内存的app的生命周期或系统事件。

举个例子，你可以实现`onTrimMemory()`回调来响应各种内存相关的时间，如下所示：

```java
import android.content.ComponentCallbacks2;
// Other import statements ...

public class MainActivity extends AppCompatActivity
    implements ComponentCallbacks2 {

    // Other activity code ...

    /**
     * Release memory when the UI becomes hidden or when system resources become low.
     * @param level the memory-related event that was raised.
     */
    public void onTrimMemory(int level) {

        // Determine which lifecycle or system event was raised.
        switch (level) {

            case ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN:

                /*
                   Release any UI objects that currently hold memory.

                   The user interface has moved to the background.
                */

                break;

            case ComponentCallbacks2.TRIM_MEMORY_RUNNING_MODERATE:
            case ComponentCallbacks2.TRIM_MEMORY_RUNNING_LOW:
            case ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL:

                /*
                   Release any memory that your app doesn't need to run.

                   The device is running low on memory while the app is running.
                   The event raised indicates the severity of the memory-related event.
                   If the event is TRIM_MEMORY_RUNNING_CRITICAL, then the system will
                   begin killing background processes.
                */

                break;

            case ComponentCallbacks2.TRIM_MEMORY_BACKGROUND:
            case ComponentCallbacks2.TRIM_MEMORY_MODERATE:
            case ComponentCallbacks2.TRIM_MEMORY_COMPLETE:

                /*
                   Release as much memory as the process can.

                   The app is on the LRU list and the system is running low on memory.
                   The event raised indicates where the app sits within the LRU list.
                   If the event is TRIM_MEMORY_COMPLETE, the process will be one of
                   the first to be terminated.
                */

                break;

            default:
                /*
                  Release any non-critical data structures.

                  The app received an unrecognized memory level value
                  from the system. Treat this as a generic low-memory message.
                */
                break;
        }
    }
}
```

`onTrimMemory()`回调在Android 4.0时加入。对于更早期的版本，你可以使用`onLowMemory()`方法，该方法大约等同于`TRIM_MEMORY_COMPLETE`事件。

### 检查你需要使用多少内存

为了允许运行多个进程，Android 对于分配给每个app设置了一个非常硬性的规定。确切的堆的大小的限制对于不同设备是变化的，它主要基于该设备总共的可用内存究竟有多少。如果你的app已经到达了堆容量的上限并且尝试获得更多内存空间，系统会抛出一个`OutOfMemoryError`的错误。

为了避免运行时用完内存，你可以查询系统来决定在当前设备上可用多少内存。你可以通过调用`getMemoryInfo()`方法来查询这个数字。这个方法返回一个`ActivityManager.MemoryInfo`对象，该对象提供了设备当前的内存状态信息，包括可用内存，总内存和内存阈值（即系统开始杀进程的内存等级）。`ActivityManager.MemoryInfo`对象同时也暴露一个简单的布尔值`lowMemory`来告诉你是否设备当前处在低内存运行状态。

以下的代码部分展示了一个如何在应用中使用`getMemoryInfo()`方法的例子：

```java
public void doSomethingMemoryIntensive() {

    // Before doing something that requires a lot of memory,
    // check to see whether the device is in a low memory state.
    ActivityManager.MemoryInfo memoryInfo = getAvailableMemory();

    if (!memoryInfo.lowMemory) {
        // Do memory intensive work ...
    }
}

// Get a MemoryInfo object for the device's current memory status.
private ActivityManager.MemoryInfo getAvailableMemory() {
    ActivityManager activityManager = (ActivityManager) this.getSystemService(ACTIVITY_SERVICE);
    ActivityManager.MemoryInfo memoryInfo = new ActivityManager.MemoryInfo();
    activityManager.getMemoryInfo(memoryInfo);
    return memoryInfo;
}
```

## 使用更加内存高效的代码来构建

一些Android特性，Java类和代码构造倾向于使用比别人更多的内存。你可以在代码中选择更加有效的备用方案来最小化App中使用的内存。

### 谨慎使用Service

在没有必要的情况下留下一个Service运行是Android app中能犯的最严重的内存管理错误之一。如果你的App需要一个Service在后台运行，除非它需要做工作的情况下，不要让它一直保持运行。记得要在完成任务之后停止你的Service。否则，你会在不经意间导致内存泄漏。

当你开始(start)了一个Service，系统更倾向于保持运行这个运行了Service的进程。这个行为导致Service的进程非常昂贵因为由Service使用剩下的内存对于其它进程来说是不可用的。这样会造成系统中可以在LRU缓存中保存的进程数量减少，使得App效率变低。当内存紧张并且系统不能保持足够的进程来控制当前运行的所有哦Service时，它甚至会导致系统内存抖动。

你通常应该避免使用持续性的Service，因为它们会对可用内存提出持续到需求。取而代之的是，我们推荐你使用可选的实现例如`JobScheduler`。对于更多有关如何使用`JobScheduler`来规划后台进程的信息，请参考[Background Optimizations](https://developer.android.com/topic/performance/background-optimization.html)。

如果你必须使用Service，最好的限制Service寿命的方法是使用`IntentService`，它会在处理完开启它的`Intent`交给它的任务之后马上结束自己。更多信息，请参阅[Running in a Background Service](https://developer.android.com/training/run-background-service/index.html)。

### 使用优化的数据容器

一些有编程语言提供的类在移动设备上使用并没有经过优化。例如泛型`HashMap`实现会变得非常内存低效，因为每个映射都需要一个单独的入口对象。

Android Framework 包含了许多优化过的数据容器，包括`SparseArray`,`SparseBooleanArray`和`LongSparseArray`。举个例子，`SparseArray`类更高效因为它避免了系统对key(有时是value)的自动装拆箱。

如有必要，你始终可以切换到原始数组以获得非常精简的数据结构。

### 注意你的代码抽象

开发者通常简单的使用抽象作为一个良好的编码实践，因为抽象可以提升代码的灵活性和维护性。然而，抽象的代价很大：通常他们需要更多的被执行的代码，需要更多的时间以及更多的内存来使代码映射到内存上。所以，如果你的抽象没有提供一个显著的好处，你应该避免它。

### 使用轻量化的 Protobufs 来序列化数据

[Protocol buffers](https://developers.google.com/protocol-buffers/docs/overview) 是一种语言无关、平台中立的可扩展机制，由谷歌设计用于序列化结构化的数据——类似于XML，但是更小巧、快速和简单。如果您决定使用protobufs作为数据，则应始终在客户端代码中使用lite protobufs。 常规protobufs会生成极其冗长的代码，这会导致应用程序出现多种问题，例如RAM使用量增加，APK大小增加以及执行速度变慢。

更多信息，请查看 [protobuf readme](https://android.googlesource.com/platform/external/protobuf/+/master/java/README.md#installation-lite-version-with-maven)中的"Lite Version"章节。

### 避免内存抖动

如之前所述，GC 事件通常不会影响你的app的表现。但是，在很短的时间内发生的许多垃圾收集事件可能会很快耗尽您的帧时间。系统花在垃圾收集上的时间越长，处理渲染或流式传输音频等其他事情的时间就越少。通常，内存抖动可能导致大量垃圾收集事件发生。 实际上，内存抖动描述了在给定时间内发生的已分配临时对象的数量。

例如，您可以在for循环中分配多个临时对象。 或者您可以在View的`onDraw()`方法内创建新的`Paint`或`Bitmap`对象。 在这两种情况下，应用程序都会以高容量快速创建大量对象。 这些可以快速消耗年轻代中的所有可用内存，从而迫使垃圾收集事件发生。

当然，您需要在代码中找到内存抖动概率高的位置，然后才能修复它们。 为此，您应该在Android Studio中使用`Memory Profiler`。

一旦确定代码中的问题区域，请尝试减少性能关键区域内的分配数量。 考虑将事物移出内部循环或者将它们移动到基于工厂的分配结构中。

## 删除内存密集型的资源和库

代码中的某些资源和库可以在不知情的情况下吞噬内存。 APK的总体大小（包括第三方库或嵌入式资源）可能会影响应用消耗的内存量。 您可以通过从代码中删除任何冗余，不必要或臃肿的组件，资源或库来提高应用程序的内存消耗。

### 减少整体 APK 的大小

您可以通过减少应用的总体大小来显着降低应用的内存使用量。 位图大小，资源，动画帧和第三方库都可以影响APK的大小。 Android Studio和Android SDK提供了多种工具来帮助您减少资源和外部依赖项的大小。 这些工具支持现代代码缩减方法，例如R8编译。 （Android Studio 3.3及更低版本使用ProGuard代替R8编译。）

有关如何减少整体APK尺寸的详细信息，请参阅 [guide on how to reduce your app size](https://developer.android.com/topic/performance/reduce-apk-size.html).。

### 使用Dagger 2进行依赖注入

依赖注入框架可以简化您编写的代码，并提供适用于测试和其他配置更改的自适应环境。

如果您打算在应用程序中使用依赖注入框架，请考虑使用Dagger 2. Dagger不使用反射来扫描应用程序的代码。 Dagger的静态编译时实现意味着它可以在Android应用程序中使用，而无需不必要的运行时成本或内存使用。

使用反射的其他依赖注入框架倾向于通过扫描代码来注释来初始化进程。 此过程可能需要更多的CPU周期和RAM，并且可能会在应用程序启动时导致明显的延迟。

### 使用外部库时要小心

外部库代码通常不是为移动环境编写的，并且在用于移动客户端上工作时效率低下。 当您决定使用外部库时，可能需要针对移动设备优化该库。 在决定使用它之前，预先规划该工作并在代码大小和RAM占用空间方面分析库。

即使是一些移动优化库也会因实现不同而导致问题。 例如，一个库可能使用lite protobufs，而另一个库使用micro protobuf，导致应用程序中有两个不同的protobuf实现。 这可能发生在日志记录，分析，图像加载框架，缓存以及许多其他您不期望的事情的不同实现中。

尽管ProGuard可以帮助使用正确的标志删除API和资源，但它无法删除库的大型内部依赖项。 您在这些库中需要的功能可能需要较低级别的依赖项。 当你使用库中的Activity子类（它往往会有大量的依赖关系）时，这会变得特别成问题，当库使用反射时（这是常见的并且意味着你需要花费大量时间手动调整ProGuard来实现它 工作），等等。

另外，请避免在几十个功能中仅使用一个或两个功能的共享库。 您不希望引入大量您甚至不使用的代码和开销。 在考虑是否使用库时，请查找与您需要的实现非常匹配的实现。 否则，您可能决定创建自己的实现。