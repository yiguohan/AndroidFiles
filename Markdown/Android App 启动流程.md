## 1. 流程中涉及的相关信息

App启动的流程主要涉及三个进程和六个类

### 1.1 三个进程

#### 1.1.1 Launcher 进程

整个App启动流程的起点，负责接收用户点击屏幕事件，它其实就是个Activity，里面实现了点击事件，长按事件，触摸等事件，可以这么理解，把Launcher想象成一个总的Activity，屏幕上各种App的Icon就是这个Activity的Button，当点击Icon时，会从Launcher跳转到其他页面。

#### 1.1.2 SystemServer 进程

这个进程在整个Android进程中是非常重要的一个，地位和Zygote等同，它是属于Application Framework层的，Android中的所有服务，例如AMS，WindowsManager，PackageManagerService等等都是由这个SystemServer fork出来的。所以它的地位可见一斑。

#### 1.1.3 App 进程

你要启动的App所运行的进程。

### 1.2 六个大类

#### 1.2.1 AMS

ActivityManagerService，即AMS。是Android中最核心的服务之一，主要负责系统四大组件的启动、切换、调度及应用进程的管理和调度等工作，其职责与操作系统中的进程管理和调度模块相类似，因此它在Android中非常重要，它本身也是一个Binder的实现类。

#### 1.2.2 Instrumentation

监控应用程序和系统的交互

#### 1.2.3 ActivityThread

应用的入口类，通过调用main方法，开启消息循坏队列。ActivityThread所在的线程被称为主线程。

#### 1.2.4 ApplicationThread

ApplicationThread提供Binder通讯接口，AMS则通过代理调用此App进程的本地方法

#### 1.2.5 ActivityManagerProxy

AMS服务在当前进程的代理类，负责与AMS通信

#### 1.2.6 ApplicationThreadProxy

ApplicationThread在AMS服务中的代理类，负责与ApplicationThread通信

## 2.App 启动步骤

1. 启动的起点发生在Launcher Activity中，启动一个App说简单就是启动一个Activity。之前说过所有的组件的启动、切换、调度都由AMS来负责，所以第一步就是**Laucher相应用户的点击事件，通知AMS**；
2. **AMS得到Launcher的通知**，就需要相应这个通知，主要就是**新建一个Task准备启动Activity，并告诉Launcher你可以休息了(Pause)**；
3. **Launcher得到AMS让自己"休息"的消息**，那么就**直接挂起，并告诉AMS我已经Paused了**；
4. **AMS知道了Launcher已经挂起**之后，就可以放心的为新的Activity准备启动工作了，首先，APP肯定需要一个新的进程来运行，所以需要创建一个新进程，这个过程需要Zygote参与，**AMS通过Socket和Zygote协商，如果需要创建进程，那么就会fork自身，创建一个进程，新的进程会导入ActivityThread类**，这就是每个应用程序都有一个ActivityThread与之对应的原因；
5. **进程创建好后，通过调用上述的ActivityThread的main方法（这是应用程序的入口），在这里开启消息循环队列**，这也是主线程默认绑定Looper的原因；
6. 这时候，App还没有启动完，要永远记住，四大组件都需要AMS去启动，**AMS将上述的应用进程信息注册到AMS中，AMS再在堆栈顶取得要启动的Activity，通过一系列链式调用去完成App启动**；

下面这张图很好的描述了上面的六个步骤：

![img](https://img-blog.csdn.net/20180309004634352?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcGdnX2NvbGQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

## 3.启动流程分析

### 3.1 Launcher调用startActivitySafely方法

点击Launcher上的微信图标时，会调用startActivitySafely方法，intent中携带微信的关键信息也就是我们在配置文件中配置的默认启动页信息，其实在微信安装的时候，Launcher已经将启动信息记录下来了，图标只是一个快捷方式的入口。

startActivitySafely的方法最终还是会调用Activity的startActivity方法

```java
@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        // Note we want to go through this call for compatibility with
        // applications that may have overridden the method.
        startActivityForResult(intent, -1);
    }
}
```

而startActivity方法最终又会回到startActivityForResult方法，这里startActivityForResult的方法中code为-1，表示startActivity并不关心是否启动成功。startActivityForResult部分方法如下所示：

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
        @Nullable Bundle options) {
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                ar.getResultData());
        }
        if (requestCode >= 0) {
```

startActivityForResult方法中又会调用mInstrumentation.execStartActivity方法，我们看到其中有个参数是`mMainThread.getApplicationThread()`

关于ActivityThread曾在 深入理解Android消息机制https://blog.csdn.net/huangliniqng文章中提到过，ActivityThread是在启动APP的时候创建的，ActivityThread代表应用程序，而我们开发中常用的Application其实是ActivityThread的上下文，在开发中我们经常使用，但在Android系统中其实地位很低的。

Android的main函数就在ActivityThread中

```java
public static void main(String[] args) {
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
    SamplingProfilerIntegration.start();
 
    // CloseGuard defaults to true and can be quite spammy.  We
    // disable it here, but selectively enable it later (via
    // StrictMode) on debug builds, but using DropBox, not logs.
    CloseGuard.setEnabled(false);
 
    Environment.initForCurrentUser();
```

再回到上面方法`mMainThread.getApplicationThread()`，得到的是一个Binder对象，代表Launcher所在的App的进程，mToken实际也是一个Binder对象，代表Launcher所在的Activity。通过Instrumentation传给AMS，这样AMS就知道是谁发起的请求。

### 3.2 mInstrumentation.execStartActivity

instrumentation在测试的时候经常用到，instrumentation的官方文档：http://developer.android.com/intl/zh-cn/reference/android/app/Instrumentation.html这里不对instrumentation进行详细介绍了，我们主要接着看mInstrumentation.execStartActivity方法

```java
public Instrumentation.ActivityResult execStartActivity(Context who, IBinder contextThread, IBinder token, Activity target, Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread)contextThread;
    if(this.mActivityMonitors != null) {
        Object e = this.mSync;
        synchronized(this.mSync) {
            int N = this.mActivityMonitors.size();
 
            for(int i = 0; i < N; ++i) {
                Instrumentation.ActivityMonitor am = (Instrumentation.ActivityMonitor)this.mActivityMonitors.get(i);
                if(am.match(who, (Activity)null, intent)) {
                    ++am.mHits;
                    if(am.isBlocking()) {
                        return requestCode >= 0?am.getResult():null;
                    }
                    break;
                }
            }
        }
    }
 
    try {
        intent.setAllowFds(false);
        intent.migrateExtraStreamToClipData();
        int var16 = ActivityManagerNative.getDefault().startActivity(whoThread, intent, intent.resolveTypeIfNeeded(who.getContentResolver()), token, target != null?target.mEmbeddedID:null, requestCode, 0, (String)null, (ParcelFileDescriptor)null, options);
        checkStartActivityResult(var16, intent);
    } catch (RemoteException var14) {
        ;
    }
 
    return null;
}
```

其实这个类是一个Binder通信类，相当于IPowerManager.java就是实现了IPowerManager.aidl，我们再来看看getDefault这个函数

```java
public static IActivityManager getDefault() {
    return (IActivityManager)gDefault.get();
}
```

getDefault方法得到一个IActivityManager，它是一个实现了IInterface的接口，里面定义了四大组件的生命周期

```java
public static IActivityManager asInterface(IBinder obj) {
    if(obj == null) {
        return null;
    } else {
        IActivityManager in = (IActivityManager)obj.queryLocalInterface("android.app.IActivityManager");
        return (IActivityManager)(in != null?in:new ActivityManagerProxy(obj));
    }
}
```

最终返回一个ActivityManagerProxy对象也就是AMP，AMP就是AMS的代理对象，说到代理其实就是代理模式，关于什么是代理模式以及动态代理和静态代理的使用可以持续关注我，后面会单独写篇文章进行介绍。

AMP的startActivity方法

```java
public int startActivity(IApplicationThread caller, Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode, int startFlags, String profileFile, ParcelFileDescriptor profileFd, Bundle options) throws RemoteException {
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken("android.app.IActivityManager");
    data.writeStrongBinder(caller != null?caller.asBinder():null);
    intent.writeToParcel(data, 0);
    data.writeString(resolvedType);
    data.writeStrongBinder(resultTo);
    data.writeString(resultWho);
    data.writeInt(requestCode);
    data.writeInt(startFlags);
    data.writeString(profileFile);
    if(profileFd != null) {
        data.writeInt(1);
        profileFd.writeToParcel(data, 1);
    } else {
        data.writeInt(0);
    }
```

主要就是将数据写入AMS进程，等待AMS的返回结果，这个过程是比较枯燥的，因为我们做插件化的时候只能对客户端Hook，而不能对服务端操作，所以我们只能静静的看着。（温馨提示：如果文章到这儿你已经有点头晕了，那就对了,研究源码主要就是梳理整个流程，千万不要纠结源码细节，那样会无法自拔）。

### 3. AMS处理Launcher的信息

AMS告诉Launcher我知道了，那么AMS如何告诉Launcher呢？

Binder的通信是平等的，谁发消息谁就是客户端，接收的一方就是服务端，前面已经将Launcher所在的进程传过来了，AMS将其保存为一个ActivityRecord对象，这个对象中有一个ApplicationThreadProxy即Binder的代理对象，AMS通ApplicationTreadProxy发送消息，App通过ApplicationThread来接收这个消息。

Launcher收到消息后，再告诉AMS，好的我知道了，那我走了，ApplicationThread调用ActivityThread的sendMessage方法给Launcher主线程发送一个消息。这个时候AMS去启动一个新的进程，并且创建ActivityThread，指定main函数入口。

启动新进程的时候为进程创建了ActivityThread对象，这个就是UI线程，进入main函数后，创建一个Looper，也就是mainLooper，并且创建Application，所以说Application只是对开发人员来说重要而已。创建好后告诉AMS微信启动好了，AMS就记录了这个APP的登记信息，以后AMS通过这个ActivityThread向APP发送消息。

这个时候AMS根据之前的记录告诉APP应该启动哪个Activity，APP就可以启动了。

```java
public void handleMessage(Message msg) {
    ActivityThread.ActivityClientRecord data;
    switch(msg.what) {
    case 100:
        Trace.traceBegin(64L, "activityStart");
        data = (ActivityThread.ActivityClientRecord)msg.obj;
        data.packageInfo = ActivityThread.this.getPackageInfoNoCheck(data.activityInfo.applicationInfo, data.compatInfo);
        ActivityThread.this.handleLaunchActivity(data, (Intent)null);
        Trace.traceEnd(64L);
```

`ActivityThread.ActivityClientRecord`就是AMS传过来的Activity

`data.activityInfo.applicationInfo`所得到的属性我们称之为LoadedApk，可以提取到apk中的所有资源，那么APP内部是如何页面跳转的呢，比如我们从ActivityA跳转到ActivityB，我们可以将Activity看作Launcher，唯一不同的就是，在正常情况下ActivityB和ActivityA所在同一进程，所以不会去创建新的进程。