---
title: App启动流程
date: 2018-02-24
tags: ["android", "app"]
---

## 前言

作为 Android 用户，我们会点击桌面图标，然后一个 app 就启动了了，在这个过程中到底发生了什么呢？换句话说 Android app 是怎么启动的呢？如果和我有一样的好奇，我们来一起走进代码的细节中了解一下。

## 桌面 app

我们平时都不会管 Framework 层，自然也不会去写桌面 app。那我给大家看一看系统带的桌面`Activity`是怎么启动我们的 Activity 的.

```java
Intent intent = new Intent(mIntent);
ListItem item = mActivitiesList.get(position);
intent.setClassName(item.packageName, item.className);
if (item.extras != null) {
    intent.putExtras(item.extras);
}
startActivity(intent);
```

原来也是调用`startActivity`，不过是隐式调用，通过包名和类名来启动对应的`Activity`，可是，这时候还没有`Application`呢，怎么能启动`Activity`呢？看过我前一篇文章的读者们会想到，这个过程中启动了`Application`，那我们就来看看这是如何启动的。

## 启动 Application

上篇 Activity 启动流程中的文章中提到，在流程图中，有一个方法很重要，它是`startSpecificActivityLocked`，因为上一篇讲的是`Activity`启动流程，所以这里和 App 启动相关的地方就略过没讲，既然上一篇画了流程图，这一篇我们就细细讲一下 App 的启动。

先看一下`startSpecificActivityLocked`引出的启动 App 进程。
这个方法我就不贴代码了，简单来说，如果进程存在，就启动 Activity，若不存在，调用下面的方法启动进程。我们就直接讲启动进程的方法。

### ActivityManagerService startProcessLocked

```java
boolean isActivityProcess = (entryPoint == null);
if (entryPoint == null) entryPoint = "android.app.ActivityThread";

Process.ProcessStartResult startResult = Process.start(entryPoint,
        app.processName, uid, uid, gids, debugFlags, mountExternal,
        app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
        app.info.dataDir, entryPointArgs);
```

这里是这个方法的核心过程， 调用`Process.start`方法启动一个进程。

看到这里，相信能解决不少开发者拥有的共同疑惑：
为什么看到某些博客会说`ActivityThread`的`main`方法是 application 启动的时候调用的`main`方法呢？
原来调用`Process.start`的时候指定了 entryPoint 为`android.app.ActivityThread`，所以，这里启动之后就会调用`ActivityThread`的`main`方法。

这里就像我们平时写 Java 程序一样，启动一个程序，会分配一个单独的 Java Runtime，从`main`方法开始执行。

那么我们就继续跟入，`Process.start`就是调用了`startViaZygote`方法，这个方法的注释是

```java
// Starts a new process via the zygote mechanism
通过zygote机制启动新进程
```

这个方法忽视掉增加参数的过程之后，这里也就一个函数

```java
return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
```

看看函数名`openZygoteSocketIfNeeded`，`openZygoteSocket`，嗯，代码跟进去就像函数名说的，通过 Socket 和 Zygote 交互，将需要的参数通过 socket 传给 Zygote，它就会创建好我们需要的 App 进程。
这样就返回了一个新进程，`ActivityThread`的`main`方法就要开始执行了。

### ActivityThread main

```java
public static void main(String[] args) {
    Environment.initForCurrentUser();

    AndroidKeyStoreProvider.install();
    // Make sure TrustedCertificateStore looks in the right place for CA certificates
    final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
    TrustedCertificateStore.setDefaultUserDirectory(configDir);
    Process.setArgV0("<pre-initialized>");
    Looper.prepareMainLooper(); // 初始化不允许退出的MainLoop
    ActivityThread thread = new ActivityThread();
    thread.attach(false);
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    // 使用liunx epoll开始死循环等待消息
    // 大家有兴趣的话看MessageQueue的nativePollOnce方法
    // 对应的c++代码 android_os_MessageQueue.cpp
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

这个代码相信大家在很多博客都见过，在这篇文章中不重要的代码解析放在了代码的注释中。
下面我们来看看这个重要的方法
`thread.attach(false);`

### ActivityThread attach

```java
final IActivityManager mgr = ActivityManagerNative.getDefault();
try {
    mgr.attachApplication(mAppThread);
} catch (RemoteException ex) {
    // Ignore
}
// Watch for getting close to heap limit.
BinderInternal.addGcWatcher(new Runnable() {
    @Override public void run() {
        if (!mSomeActivitiesChanged) {
            return;
        }
        Runtime runtime = Runtime.getRuntime();
        long dalvikMax = runtime.maxMemory();
        long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
        if (dalvikUsed > ((3*dalvikMax)/4)) {
            if (DEBUG_MEMORY_TRIM) Slog.d(TAG, "Dalvik max=" + (dalvikMax/1024)
                    + " total=" + (runtime.totalMemory()/1024)
                    + " used=" + (dalvikUsed/1024));
            mSomeActivitiesChanged = false;
            try {
                mgr.releaseSomeActivities(mAppThread);
            } catch (RemoteException e) {
            }
        }
    }
});
```

这里重要方法就是`mgr.attachApplication(mAppThread);`，但是多复制了一些代码，为了提一下 GC，什么时候虚拟机进行 GC 呢？这里给了我们答案，在使用超过 3/4 最大内存的时候。

继续回来跟代码，和前面一样，这里使用的是`ActivityManagerService`作为`IActivityManager`，
`mgr.attachApplication(mAppThread)`这个方法就是调用了`attachApplicationLocked`方法。

```java
 ensurePackageDexOpt(app.instrumentationInfo != null
        ? app.instrumentationInfo.packageName
        : app.info.packageName);
if (app.instrumentationClass != null) {
    ensurePackageDexOpt(app.instrumentationClass.getPackageName());
}

thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
        profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
        app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
        isRestrictedBackupMode || !normalMode, app.persistent,
        new Configuration(mConfiguration), app.compat,
        getCommonServicesLocked(app.isolated),
        mCoreSettingsObserver.getCoreSettingsLocked());
```

这里主要是两段，第一段就是单纯为了解惑 dexOpt 是什么时候生成的呢？就是这个时候。

后面是一个方法: `ActivityThread`的`bindApplication`方法。

在忽视`bindApplication`方法中的一些初始化之后，就做了一件事，给`H`这个`Handler`发送`BIND_APPLICATION`消息，然后`H`会调用`handleBindApplication`方法。

### ActivityThread handleBindApplication

```java
// If the app is Honeycomb MR1 or earlier, switch its AsyncTask
// implementation to use the pool executor.  Normally, we use the
// serialized executor as the default. This has to happen in the
// main thread so the main looper is set right.
if (data.appInfo.targetSdkVersion <= android.os.Build.VERSION_CODES.HONEYCOMB_MR1) {
    AsyncTask.setDefaultExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
}
Message.updateCheckRecycle(data.appInfo.targetSdkVersion);

if ((data.appInfo.flags&ApplicationInfo.FLAG_LARGE_HEAP) != 0) {
    dalvik.system.VMRuntime.getRuntime().clearGrowthLimit();
} else {
    // Small heap, clamp to the current growth limit and let the heap release
    // pages after the growth limit to the non growth limit capacity. b/18387825
    dalvik.system.VMRuntime.getRuntime().clampGrowthLimit();
}

 // a restricted environment with the base application class.
Application app = data.info.makeApplication(data.restrictedBackupMode, null);

List<ProviderInfo> providers = data.providers;
if (providers != null) {
    installContentProviders(app, providers);
    // For process that contains content providers, we want to
    // ensure that the JIT is enabled "at some point".
    mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);
}

mInstrumentation.callApplicationOnCreate(app);
```

这里放这些代码，但是只会详解生成`Application`的函数:
`data.info.makeApplication(data.restrictedBackupMode, null)`。

剩下的我们就提以下几点:

- `AsyncTask`在不同的`targetSdkVersion`中行为不同，为什么呢，不止 AsyncTask 代码的不同，还因为这里修改了线程池的实现。
- `targetSdkVersion`是 app 运行时拿到的 sdk 版本信息之一，另一个是`minSdkVersion`，所以，Google 对 app 的限制就全加在了`targetSdkVersion`上。
- 有些 app 会设置`FLAG_LARGE_HEAP`属性，这个属性具体做了什么呢？从这里就是线索。
- `ContentProvider`的初始化原来是在`Application`的`onCreate`之前`attachBaseContext`之后。
- `Application`的生命周期也是由`Instrumentation`来控制的

看完了上面这几点，我们详细看一下生成`Application`的函数:

### LoadedApk makeApplication

```java
String appClass = mApplicationInfo.className;
if (forceDefaultAppClass || (appClass == null)) {
    appClass = "android.app.Application";
}

java.lang.ClassLoader cl = getClassLoader();

ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
app = mActivityThread.mInstrumentation.newApplication(
        cl, appClass, appContext);
```

这就是我们需要关注的这个方法的内容。

你没有声明自己的`Application`的话，将会使用系统默认的`android.app.Application`。

`Application`的初始化依旧是反射，通过`clazz.newInstance()`来生成，这也就说明了，为什么我们不能写带参数的构造函数。

其次，生成`Application`的时候还会回调`attachBaseContext`方法，这里也就成为了一个 App 最早执行代码的位置。这样，一个`Application`就生成并启动了。

到这里，`ActivityThread`的`bindApplication`方法就告一段落了。也就意味着整个`Application`的启动完成了。

## 总结

这样，我们就大概了解了一个 App 从点击桌面 icon，到`Application`完全启动的过程。

了解这个过程，首先让我们对 Manifest 中各种属性有了更深的了解；其次，了解了 Android 代码的执行过程，以写 Java 程序的视角来看 Android 程序就不再那么奇怪了；最后，有了对源码的了解，你才能写出更好的代码。
