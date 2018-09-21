---
title: Activity启动流程
date: 2018-01-21
tags: 
  - android
  - activity
---

## 前言

作为Android开发，我们经常会写`startActivity`方法，这样我们就可以启动`Activity`了，在这个过程中到底发生了什么呢？换句话说`Activity`是怎么启动的呢？如果和我有一样的好奇，我们来一起走进代码的细节中了解一下。

## 启动Activity

查看源码可以知道，`startActivity()`方法最终是调用了`startActivityForResult()`方法来执行的。

### startActivityForResult

~~~ java
Instrumentation.ActivityResult ar =
    mInstrumentation.execStartActivity(
        this, mMainThread.getApplicationThread(), mToken, this,
        intent, requestCode, options);
if (ar != null) {
    mMainThread.sendActivityResult(
        mToken, mEmbeddedID, requestCode, ar.getResultCode(),
        ar.getResultData());
}
~~~

这里的主要就是这个方法:`mInstrumentation.execStartActivity()`。

这里的流程很复杂，我们来看流程图。

![generate_activity.png](/img/activity/generate_activity.png)

在最后的`handleLaunchActivity`方法中，调用了`performLaunchActivity`生成`Activity`，
调用`handleResumeActivity`让`Activity`进入Resume生命周期。

这里也留下一个伏笔，后面写Android App的启动流程的时候还是在这个流程中，尤其是`startSpecificActivityLocked`方法。

### performLaunchActivity

~~~ java
java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent)
~~~

生成`Activity`的核心就这两句代码，通过`Instrumentation`的`newActivity`方法生成。

这里要记住`Instrumentation`，它很重要，但是先放着，我们继续来看`Activity`的启动流程。

### callActivityOnResume

`handleResumeActivity`经过多层调用，最终调用了`Instrumentation`的`callActivityOnResume`方法

~~~ java
 public void callActivityOnResume(Activity activity) {
    activity.mResumed = true;
    activity.onResume();
}
~~~

这样`Activity`的`onResume`方法就会被调用。也就意味着`Activity`生命周期走到Resume的状态，这样的`Activity`算是启动完成了。

## 总结

我们可以看到，从`startActivity`到`Activity`进入Resume的生命周期，有一个类的身影若隐若现，那就是`Instrumentation`, 这个类负责管理`Application`的创建和启动，负责`Activity`的创建和生命周期的管理。所以，一旦你操纵这个类，你能做的事情就有很多。比如，在`Activity`生命周期的过程中添加一些消息。

经过我的总结，Activity的生命周期是有一个套路在的，这个套路就是:

> scheduleXXXActivity->handleXXXActivity->performXXActivity->CallActivitiyXXX—>Activity.XXX

`scheduleXXXActivity`负责发消息给`Handler(H)`发对应的消息。
接着就和之前文章中提到的内容类似，每个生命周期都要经过这样一个过程来调度。

到这里，我们就大概浏览了一下`Activity`的启动流程，希望你在遇到这方面的问题的时候，能把这个博客当作一个目录，知道去哪里找对应的源码，思考对应的解决方案。