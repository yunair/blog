---
title: (翻译)什么是Context
date: 2016-01-09
tags: ["翻译"]
---

`原文 ： https://possiblemobile.com/2013/06/context/`

### Context 大概是 Android 应用中使用最多的元素，同时，它可能也是最被滥用的元素

Context 对象如此常见，经常在不同的类中被传来传去，它可以用于你完全不关心的场景，如加载资源，启动新的 Activity，获取系统服务，获取内部文件路径，创建所有需要 Context 的视图(View)来完成任务，更关键的是，这还仅仅是 Context 的一小部分功能。我希望这篇文章能帮助你，在你的应用在你的应用中，更有效的使用 Context 对象。

### Context 类型

不是所有的 Context 都是相同的。获取的 Context 是由 Android 系统组件决定的，每个组件获取的 Context 对象略有不同。

**Application** — 在你的应用进程中，它是一个单例。 它可以通过`Activity`和`Service`中的*getApplication()* 方法获取，也可以通过任何继承了 Context 对象中的*getApplicationContext()*  方法，无论从哪里获得`Application`对象，只要在应用进程中，你获得的都是同一个实例。

**Activity/Service** — （这里前半部分我不按原文翻译，因为最新的 Android 源码已经不像作者写文章的时候那样了。翻译会误导）都继承了*ContextWrapper*对象，*ContextWrapper*就像命名的那样，它的内部包裹了一个 Context，它其中的方法其实就是*Context*的方法。当系统创建一个新的`Activity`或`Service`实例的时候，它同样创建了一个新的*ContextImpl*实例来完成所有费时的操作。对于每一个`Activity`或`Service`实例对应一个独一无二的*Context*。

**BroadcastReceiver** — 它本身不是*Context*，但是系统会在每次广播事件到达时，给它的*onReceive()*传入一个*Context*。这个*Context*实例是一个*ReceiverRestrictedContext* ，它的两个方法不建议使用，分别是*registerReceiver()*  和  *bindService()*。每次一个接收者接收到一个广播，则会新生成一个*Context*。

**ContentProvider** — 同样不是一个*Context*，但是当它创建的时候会传给它一个*Context*对象，然后通过它的*getContext()*方法来访问。如果`ContentProvider`和调用者在同一个应用进程中，则它会返回相同的`Application`实例，如果两个在不同的进程中，它将会新创建一个实例，代表这个`ContentProvider`所在的包。

## 保存 Context 引用

我们需要指出的第一个问题来自于在一个对象或一个类中保存  *Context*  的引用，但是这个对象存在的时间超过了保存的拥有生命周期的  *Context*  的实例的存在时间（其实就是一个某个对象持有拥有声明周期的  *Context*  对象，生命周期结束，但那个类还继续持有  *Context*  对象，这样就造成了内存泄漏）。例如，创建一个自定义的单例，需要一个  *Context*  来读取资源或者使用`ContentProvider`，然后，在那个单例中保存了当前的`Activity`或`Service`。

**Bad Singleton**

```java
public class CustomManager {
    private static CustomManager sInstance;

    public static CustomManager getInstance(Context context) {
        if (sInstance == null) {
            sInstance = new CustomManager(context);
        }

        return sInstance;
    }

    private Context mContext;

    private CustomManager(Context context) {
        mContext = context;
    }
}
```

这里的问题是我们不知道  *Context*从哪里来，同时，当`Activity`或`Service`的生命周期结束时，依旧持有它的引用是不安全的。因为单例在封闭的类中被单独的静态引用管理。这意味着我们的对象，以及所有其他被它引用的对象将不会被垃圾回收器回收。如果  *Context*是一个`Activity`，我们将会在内存中保存它所有的视图以及和它有关系的潜在的大对象；这样将会导致内存泄漏。

为了避免这样，我们修改单例，通过引用`Application`对象的  *Context*：

**Better Singleton**

```java
public class CustomManager {
    private static CustomManager sInstance;

    public static CustomManager getInstance(Context context) {
        if (sInstance == null) {
            //Always pass in the Application Context
            sInstance = new CustomManager(context.getApplicationContext());
        }

        return sInstance;
    }

    private Context mContext;

    private CustomManager(Context context) {
        mContext = context;
    }
}
```

现在，我们不用关心这个 context 来自哪里了，因为我们持有的引用是安全的。`Application`对象的  *Context*本身就是一个单例，所以即使有其他的静态对象引用它也不会造成任何泄漏。另外一个很好的可能偶尔发生的例子是：在正在运行的后台线程或者 Pending(将要发生的) *Handler*中，保存了  *Context*  的引用。

那么，为什么我们不能总是引用`Application`对象的  *Context*？就像我们前面提到的一样，以中间人的方式（指`context.getApplicationContext()`）实现，并且不需要担心内存泄漏？答案是像我在前言中暗示的一样，因为*Context*的类别不一样。

## Context 能力

依赖于  *Context*的来源，你可以安全的对给予的  *Context*对象做一些共同的操作。下面的表格说明了应用在常用的地方收到一个  *Context*，在每个情况下， *Context*对象可以用来做什么

|                            | Application | Activity | Service | ContentProvider | BroadcastReceiver |
| -------------------------- | :---------: | :------: | :-----: | :-------------: | :---------------: |
| Show a Dialog              |     NO      |   YES    |   NO    |       NO        |        NO         |
| Start an Activity          |   NO[^1]    |   YES    | NO[^1]  |     NO[^1]      |      NO[^1]       |
| Layout Inflation           |   NO[^2]    |   YES    | NO[^2]  |     NO[^2]      |      NO[^2]       |
| Start a Service            |     YES     |   YES    |   YES   |       YES       |        YES        |
| Bind to a Service          |     YES     |   YES    |   YES   |       YES       |        NO         |
| Send a Broadcast           |     YES     |   YES    |   YES   |       YES       |        YES        |
| Register BroadcastReceiver |     YES     |   YES    |   YES   |       YES       |      NO[^3]       |
| Load Resource Values       |     YES     |   YES    |   YES   |       YES       |        YES        |

1. 在这里，一个`Application`的  *Context*  可以启动一个`Activity`，但是它需要创建一个新的任务栈。在某些情况下，这可能是很好的使用方式，但是，在你的应用中，这将会导致回退栈的行为不像标准的那样，同时一般来说它不会被推荐或者被认为是一个好习惯。

2. 绘制布局(Inflation Layout，在本文中把 Inflation 翻译为绘制，下同)是合法的，但是绘制的时候会使用正在运行系统的默认主题，而不会使用在你应用中定义的主题(theme)。

3. 当`BroadcastReceiver`是*null*的时候是允许的，在 Android 4.2 及以上版本，它被用于获取 sticky 广播(broadcast)当前的值。

   ​

## 用户界面

从上面的表格可以看出，对于`Application`的  *Context*，有一些方法并不适用；那些方法都是和 UI 相关的。实际上，唯一一个有能力处理所有和 UI 相关情况的组件是`Activity`，在其它的方法中，所有组件的功能基本都相同。

幸运的是，在整个应用了，除了`Activity`之外的领域基本不使用这三个方法，这个应该是 Android 系统的设计者刻意为之。尝试使用`Application`的  *Context*显示一个  *Dialog*  或者启动一个`Activity`，系统将会抛出一个异常同时你的应用将会崩溃——明确的提示你有些地方出现错误。

绘制布局的问题则不明显。如果你读过我上一篇文章[layout inflation][1]([翻译][2])， 你应该知道在一些隐藏的行为中会做一些神秘的处理；使用正确的  *Context*将会得到相同的行为(using the right *Context* is linked to another one of those behaviors 这句话自我感觉翻译不好，放原文在此)，使用`Application`的  *Context*创建*LayoutInflater*  对象然后 Inflation Layout，对于这样调用此方法，虽然 Android 系统不会发出警告，而且这个方法将会完美返回正确的视图层级(view hierarchy)，但是在此过程中，你应用的主题(themes)和风格(styles)不会被使用。这是因为在你的清单文件中(manifest)，你定义的主题(themes)只会对`Activity`的  *Context*  起作用。任何其他的  *Context*  实例在绘制你的视图的时候将会使用系统默认主题，这将会导致最终的显示效果不如你期望那样。

## 这些规则的交集

不变的是，有人会觉得这两个规则是冲突的。举个例子，你应用当前的设计是：必须保存一个长时间的

*Context*引用，同时它必须是`Activity`的  *Context*，因为我们要用它完成的任务包括操作 UI。如果在那种情况，我强烈建议你重新思考你的设计。因为这将是教科书式的反系统设计。

## 经验之谈

在大多数情况下，在你封闭的组件中可以直接使用可用的*Context*对象。你可以安全的保存它的引用只要那个组件在生命周期结束后不再存在。只要你的对象需要保存的*Context*引用，并且你的对象存活的时间超出了`Activity`或`Service`的生命周期，即使这种情况偶尔出现，请使用`Application`的  *Context*。

[1]: http://www.doubleencore.com/2013/05/layout-inflation-as-intended/
[2]: http://yunair.github.io/blog/2016/01/12/%5B%E7%BF%BB%E8%AF%91%5D%E6%8C%89%E8%AE%A1%E5%88%92%E7%BB%98%E5%88%B6%E5%B8%83%E5%B1%80.html
