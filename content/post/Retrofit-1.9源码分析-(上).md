---
title: Retrofit-1.9源码分析-(上)
date: 2016-01-13
tags: ["retrofit", "square"]
---

[Retrofit 项目的 GitHub][1]

Retrofit 官网介绍 Type-safe HTTP client for Android and Java。

接下来的这几篇文章的分析都是基于 1.9 版本的。

看过我前面文章的开发者们应该知道，我不喜欢把一篇文章写的很长，所以这个分析依旧会有好几篇，好了，我们开始第一篇的分析。

看到一个库，我们要首先会用，然后再去深入其中，研究其中的源代码。

那我们来看看这个库的用法。

### 用法

官网上的教程是这样写的。

首先定义请求接口：

```java
public interface GitHubService {
    //被注释的内容是同步方法，在Android 4.0以后不允许主线程进行网络请求，所以一般不这么用
    //@GET("/users/{user}/repos")
    //List<Repo> listRepos(@Path("user") String user);
    @GET("/users/{user}/repos")
    void listRepos(@Path("user") String user, Callback<List<Repo>> callback);
}
```

然后初始化`RestAdapter`，并且通过 RestAdapter 生成刚才代理的实现类

```java
RestAdapter restAdapter = new RestAdapter.Builder()
    .setEndpoint("https://api.github.com")
    .build();

GitHubService service = restAdapter.create(GitHubService.class);
```

现在就可以调用接口发送请求了。

```java
service.listRepos("octocat", new Callback<List<Repo>>(){
    @Override
    public void success(List<Repo> repos, Response response) {
    }

    @Override
    public void failure(RetrofitError error) {
    }
});
```

这样就返回了网络请求对应的 Model，并且通过回调让返回的结果在主线程进行操作，

从网络请求返回的数据到对应的 Model 转换是在`RestAdapter`那里定义的，默认是`Gson`解析，即默认服务端返回格式是 Json，

当然你也可以使用不一样的 Json 解析库，当服务端返回 Xml，Protocol Buffer 的时候，你也可以解析，只要自己实现 Converter 接口，并在`RestAdapter`中设置即可。

这里有一个问题，如果服务端的接口不是全部返回一个格式怎么办？或者不是同一个 Endpoint 怎么办？每一个都需要定义一个对应的`RestAdapter`对应。

所以这里可能出现的问题就是，某些接口可能数据很简单，但是返回格式不统一，那么就需要新写一个这样类似的模板代码。

这样，我们大概了解了这个版本的库使用方式：

1. 定义一个接口，通过注解说明它是什么请求类型，并且在注解内写明它的相对于 Endpoint(2.0 版本称作 BaseUrl,这样就好理解多了)的 Url，

   然后声明一个方法，这个方法中的参数就是网络请求时服务端需要的参数，也可以把网络请求的路径作为方法参数，最后定义一个回调，让返回结果在主线程中执行。

2. 定义`RestAdapter`，并对其进行必要的设置，通过它生成那个接口对应的实现类。

3. 调用实现类，执行网络请求。

可以看出，定义`RestAdapter`是其中很重要的一步，那么我们先来分析它到底做了什么。

### 初始化 RestAdapter

我们看到示例，通过`Builder`来初始化的，那么源码中一定用了建造着模式，一般情况下，如果可设置项不多，不会用这样一个模式，

看来`RestAdapter`的可设置项非常多，我们来看看。

在`RestAdapter`内部的静态`Builder`类中，有这么多可以设置的内容。

#### RestAdapter.Builder

```java
    private Endpoint endpoint;  //就是BaseUrl，
    private Client.Provider clientProvider; //最终执行网络请求使用的库
    private Executor httpExecutor;  //http线程池的具体实现
    private Executor callbackExecutor; //callback在哪个线程执行
    private RequestInterceptor requestInterceptor; //网络请求的拦截者，可以给网络请求加入共同的参数
    private Converter converter; //网络请求返回值的解析方式(Json/Xml)
    private Profiler profiler;   //可以记录HTTP方法执行时间和状态码
    private ErrorHandler errorHandler; //错误处理
    private Log log;
    private LogLevel logLevel = LogLevel.NONE;
```

如果分别深入每个类，那么，除了最后的 LogLevel 是 enum 之外，其他都是接口，有多少开发者还记得面向接口编程，不面向实现编程，

或者说作为开发者的我们记得这句话，但是自己实现的时候都忘记了。

其实这些接口就是一个个 hook，留给使用者去自定义。但是这个库也提供了默认的参数，那这个默认的参数在哪里设置呢？

看回我们前面的例子，对于建造着模式，最后一步要使用`build()`方法来返回一个对应的类，那么我们来看一下这个`build()`方法。

```java
/** Create the {@link RestAdapter} instances. */
    public RestAdapter build() {
      if (endpoint == null) {
        throw new IllegalArgumentException("Endpoint may not be null.");
      }
      ensureSaneDefaults();
      return new RestAdapter(endpoint, clientProvider, httpExecutor, callbackExecutor,
          requestInterceptor, converter, profiler, errorHandler, log, logLevel);
    }
```

这里很容易可以看出，必须要设置 EndPoint 才能继续使用。

那么其它的类变量一定在`ensureSaneDefaults()`方法初始化了，否则当库的使用者未设置的时候，其它都是`null`了。

我们来看一下它的源码：

#### RestAdapter ensureSaneDefaults()

```java
    private void ensureSaneDefaults() {
      if (converter == null) {
        converter = Platform.get().defaultConverter();
      }
      if (clientProvider == null) {
        clientProvider = Platform.get().defaultClient();
      }
      if (httpExecutor == null) {
        httpExecutor = Platform.get().defaultHttpExecutor();
      }
      if (callbackExecutor == null) {
        callbackExecutor = Platform.get().defaultCallbackExecutor();
      }
      if (errorHandler == null) {
        errorHandler = ErrorHandler.DEFAULT;
      }
      if (log == null) {
        log = Platform.get().defaultLog();
      }
      if (requestInterceptor == null) {
        requestInterceptor = RequestInterceptor.NONE;
      }
    }
  }
```

可以看到，主要需要关心的接口都是由`Platform`类实现的，那我们进入其中一探究竟。

#### Platform

```java
abstract class Platform{
  private static final Platform PLATFORM = findPlatform();

  static Platform get() {
    return PLATFORM;
  }

  private static Platform findPlatform() {
      try {
        Class.forName("android.os.Build");
        if (Build.VERSION.SDK_INT != 0) {
          return new Android();
        }
      } catch (ClassNotFoundException ignored) {
      }

      if (System.getProperty("com.google.appengine.runtime.version") != null) {
        return new AppEngine();
      }

      return new Base();
    }
}
```

想要初始化它们，先使用`Platform.get()`方法，这个方法获取了单例`Platform`对象 PLATFORM，这里给了我们实现单例的一种思路。

通过`findPlatform()`方法，如果是 Android 平台，则使用内部的`Android`子类，如果是 Google AppEngine，则使用内部的`AppEngine`子类，

如果都不是，则使用内部的`Base`子类，还记得这个库的目标吧？是 Java 和 Android 平台的 HTTP 库，所以这里还可能是 Java 平台，所以要使用另外一种实现方式。

这次分析主要针对 Android 平台，所以如果好奇另外两个实现，请自己查看源码。

还记得我们在使用`RestAdapter`的`Builder`的时候除了`Endpoint`之外的接口吗？在`Platform`内部，定义了其它默认的实现：

```java
abstract Converter defaultConverter();
abstract Client.Provider defaultClient();
abstract Executor defaultHttpExecutor();
abstract Executor defaultCallbackExecutor();
abstract RestAdapter.Log defaultLog();
```

均是抽象方法，意味着这些都要靠子类实现，那么我们看一下`Android`子类是如何实现的：

#### Platform.Android

```java
private static class Android extends Platform {
    @Override Converter defaultConverter() {
      return new GsonConverter(new Gson()); //默认Gson，处理请求返回的Json
    }

    @Override Client.Provider defaultClient() {
      final Client client;
      if (hasOkHttpOnClasspath()) {     //存在Okhttp库，当然用OkHttp了
        client = OkClientInstantiator.instantiate();
      } else if (Build.VERSION.SDK_INT < Build.VERSION_CODES.GINGERBREAD) {
        client = new AndroidApacheClient();  //Android 2.3以下用HttpClient
      } else {
        client = new UrlConnectionClient(); //Android 2.3以上用HttpUrlConnection
      }
      return new Client.Provider() {
        @Override public Client get() {
          return client;
        }
      };
    }

    @Override Executor defaultHttpExecutor() {
      return Executors.newCachedThreadPool(new ThreadFactory() { //使用CachedThreadPool来管理网络请求
        @Override public Thread newThread(final Runnable r) {
          return new Thread(new Runnable() {
            @Override public void run() {
              Process.setThreadPriority(THREAD_PRIORITY_BACKGROUND); //给线程设置优先级
              r.run();
            }
          }, RestAdapter.IDLE_THREAD_NAME);
        }
      });
    }

    @Override Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();  //callback在主线程执行
    }

    @Override RestAdapter.Log defaultLog() {
      return new AndroidLog("Retrofit");
    }
  }
```

这里`hasOkHttpOnClasspath()`方法我觉得很棒，和前面找对应的平台一样，通过`Class.forName(包名)`找类，看是否存在。

如果自己写一个库的时候是好用的一种方式，支持一些新的东东。放一下它的源码。

#### Platform hasOkHttpOnClasspath()

```java
private static boolean hasOkHttpOnClasspath() {
    try {
      Class.forName("com.squareup.okhttp.OkHttpClient");
      return true;
    } catch (ClassNotFoundException ignored) {
    }
    return false;
}
```

到这里为止，`RestAdapter`就创建好了，接下来跟着例子继续，`RestAdapter`调用`create()`方法，创建接口的实现类。

### RestAdapter create()

```java
public <T> T create(Class<T> service) {
    Utils.validateServiceClass(service); //验证Service是否有效，判断条件：1.这个类是接口  2.该接口不是其它接口的子接口 (接口也可以继承)
    return (T) Proxy.newProxyInstance(service.getClassLoader(),
				new Class<?>[] { service },
				new RestHandler(getMethodInfoCache(service)));
}
```

这里，通过一个动态代理生成了接口的实现类，并在第三个参数里做了一些的操作。

\*\*动态代理的作用：在运行时创建一个实现了一组给定接口的新类。

为什么要用动态代理？因为在编译时无法确定需要实现哪个接口。\*\*

我们慢慢分析，先看这个方法的签名:

```java
/**
   * Returns an instance of the dynamically built class for the specified
   * interfaces. Method invocations on the returned instance are forwarded to
   * the specified invocation handler. The interfaces must be visible from the
   * supplied class loader; no duplicates are permitted. All non-public
   * interfaces must be defined in the same package.
   *
   * @param loader
   *            the class loader that will define the proxy class
   * @param interfaces
   *            an array of {@code Class} objects, each one identifying an
   *            interface thill be implemented by the returned proxy
   *            object
   * @param invocationHandler
   *            the invocation handler that handles the dispatched method
   *            invocations
   * @return a new proxy object that delegates to the handler {@code h}
**/
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces,
			InvocationHandler invocationHandler)throws IllegalArgumentException {
	return getProxyClass(loader, interfaces)
      .getConstructor(InvocationHandler.class)
      .newInstance(invocationHandler);
}
```

方法体不管它的错误处理部分，先看它的参数，这个方法有三个参数：

1. loader，没什么可说的，就是一个 ClassLoader
2. interfaces, 传入的接口数组，里面的方法由代理实现
3. invocationHandler, 方法调用的处理器

对于前两个参数默认大家都已经熟悉了，我们看一下最后一个`InvocationHandler`,这个接口很简单，只定义了一个方法，如下：

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
```

举个例子来讲这个方法的含义：

拿经纪人的比喻来说吧，

简单来说，你需要让我做一件事情(要调用原始对象的一个函数)，但是，因为我不很会和别人谈判(作为框架不能和业务逻辑耦合太多)，

那么我就找了一个经纪人，由他来和你谈，当经纪人把前前后后的各种事情帮我弄好之后，

(可以过滤请求，可以让其他对象来完成对原始对象的调用)

我就可以做我的事情(原始对象调用自己的那个函数)，而不用为这些不擅长的事情分心了。

看看`invoke()`方法注释中的例子：

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    //do some processing before the method invocation

    //invoke the method
    Object result = method.invoke(proxy, args);

    //do some processing after the method invocation
    return result;
}
```

在调用方法前后都能加一些你需要做的额外工作，是不是很像我举的例子？还有很重要的一点，你要把原始对象做完的结果返回给调用者。

我们把视线移回到`newProxyInstance()`的方法体中，在这里先看一下为什么这个方法返回的对象可以被强制转换为传入的接口对象，

就像官方提供的 demo 一样，

```java
GitHubService service = restAdapter.create(GitHubService.class);
```

传入`GitHubService`，然后最后就可以强制转换为`GitHubService`对象。

\*\*原因就是在`newProxyInstance()`这个方法的第二个参数上，我们给这个代理对象提供了一组什么接口，那么我这个代理对象就会实现了这组接口，

这个时候我们当然可以将这个代理对象强制类型转化为这组接口中的任意一个。\*\*

我们传入`GitHubService`,代理类把这个接口的方法都实现了，自然能转换成`GitHubService`对象。

在这里第一篇分析就结束了，可以看到`new RestHandler(getMethodInfoCache(service))`这个方法都没有跟进去，

或许由于局限在 Java 这门语言的缘故，是因为我觉得`Proxy.newProxyInstance()`方法很不容易理解，所以到这里结束，

[1]: https://github.com/square/retrofit

​
