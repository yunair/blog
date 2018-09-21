---
title: Retrofit2源码分析-(上)
date: 2016-05-08
tags: ["retrofit", "square"]
---

### 前言

​ 千呼万唤始出来的 Retrofit2.0 正式版终于出来了，首先我们来看一看 Jake Wharton 的演讲，这次演讲主要讲了 1.0 版本的好处和问题，以及 2.0 版本的优势，对于好处，我们在 1.0 版本的使用过程中相信都已体会过了，在这里我就不重新提了，那么这第一篇就主要讲一下 1.0 版本的问题，和 2.0 在设计上是如何解决这些问题的。至于源代码的分析和对比将会放在第二篇中讲。

### Retrofit1 问题

- 无法同时获取响应返回的原始数据，比如请求头或者请求的 URL，和反序列化的响应返回的 body。
- 如果你写了一个请求，有时同步执行，有时异步执行，你就需要写两个基本一样的请求。（如果使用 RxJava 可以实现，但是你要知道如何创建 Observables）
- converter 的工作方式事实上稍微有点低效。
- 严重限制了我们把自定义类作为参数加入请求的能力。

### Retrofit2

#### Call

内置和 OkHttp 的`Call`类型一样语义的 Call 类型，不同的是它能做一些额外的事情，比如反序列化对象，相反 OkHttp 只返回原始的内容给你，这就是 Retrofit2 实现一个类似的`Call`而不是用 OkHttp 中的`Call`的原因。它是一个请求/响应对，每一个`Call`实例只能使用一次，它有 Java 的 clone 方法来创建新的实例（这个操作很廉价）。另一个巨大的好处是它把同步和异步的执行统一在了一起，这样就解决了 Retrofit1 中同步请求和异步请求要写两个的问题，同时，它可以被取消，这个是 Retrofit1 中不存在的特性。接下来我们看一下它的使用:

```java
interface GitHubService {
	@GET("/repos/{owner}/{repo}/contributors")
	Call<List<Contributor>> repoContributors(
  	@Path("owner") String owner,
  	@Path("repo") String repo);
	}
	Call<List<Contributor>> call =
	gitHubService.repoContributors("square", "retrofit")
};
```

#### 参数化 Response 对象

通过 Response，你可以获取请求的元数据，包括 response code, message, 和 headers。

```java
class Response<T> {
  int code();
  String message();
  Headers headers();

  boolean isSuccess();
  T body();
  ResponseBody errorBody();
  com.squareup.okhttp.Response raw();
}
```

只有响应的 Http code 是 200 的情况下才会进行反序列化。如果响应没有成功，我们不知道响应是什么类型。将回返回`ResponseBody`类型，它将 Content-Type，Content-Length，还有原始的 body 进行了封装，让你可以对响应做出想要的处理。

#### 动态 URL 参数

有这样一种情况，请求列表的时候通常都要给服务端传入 page，告诉服务端你要第几页数据，如果服务端将 page 相关的信息放在了响应头中，比如下一页 page 是什么，总共有几页等等，甚至服务端返回你接下来需要请求的 URL。例如 Github 的 API，Retrofit1 很难处理这种情况。我们有一个新的`@Url`注解，允许你在参数中传入 Url。如下所示:

```java
@GET
Call<List<Contributor>> repoContributorsPaginate(
      @Url String url);
```

#### 多样的，高效的 Converters

Retrofit1 有一个 Converter 的问题，当然，对于大多数人来说这都不算是问题，但对于一个库来说，它就成了问题。考虑下面一种场景，服务端有两套 API，只是返回值的类型不同，比如一个返回 json，一个返回 xml，可能是如下的 url:

```java
@GET("/some/xml/endpoint")
@GET("/some/json/endpoint")
```

在 Retrofit1 中，你不得不定义两个接口，初始化两个`RestAdapter`，因为它只有一个 Converter，每一个 Converter 和一个`RestAdapter`绑定在一起，但是，我们认为这些操作同属于一组 API，而不是分离的 API，所以，API 的返回类型不应该成为你组织`Service`的方式。

我们会按你添加的顺序来一个个的问 Converter 是否可以序列化该返回值，如果它的回答是 yes，那么就交给该 Converter 来序列化返回值。

两点需要注意的地方:

- 由于 JSON 没有任何要求或者限制，所以我们无法知道它是否可以被 JSON Converter 序列化，所以 JSON Converter 永远会回答 yes，在这种情况下，JSON Converter 应该是你添加的最后一个 Converter。
- 和 Retrofit1 不同的是，Retrofit2 本身提供内置的 Converter，但是只是提供了三种基础的 Converter，像 Json 之类的 Converter 不会内置，所以你需要明确的声明你所用的 Converter。当然，我们提供了一些 Converter，但是你需要把这些 Converter 作为独立的依赖加入项目中。

#### 多种多样插件形式的执行机制

之前，我们只有死板的执行机制。现在我们把它弄成了插件形式的，它的工作方式和 Covnerter 类似。
你可以加入你自己的方式，或者选择一个我们提供的已有的方式，我们依旧提供 RxJava 这个执行方式，但是它和 Retrofit2 库分离，需要单独的引入。

```java
interface GitHubService {
  @GET("/repos/{owner}/{repo}/contributors")
  Call<List<Contributor>> repoContributors(..);

  @GET("/repos/{owner}/{repo}/contributors")
  Observable<List<Contributor>> repoContributors2(..);

  @GET("/repos/{owner}/{repo}/contributors")
  Future<List<Contributor>> repoContributors3(..);
}
```

我们单纯的查看返回类型，对于第一个函数来说，我们对执行机制的询问过程是这样的。`call —> RxJava? No! —> Call? Yes!`

对于第三个 Future,将会抛出一个异常，因为你没有加入你的执行机制，导致了执行机制无法执行。该过程是`Future —> RxJava? No! —> Call? No! —> Throw!`。如果你想添加自己的执行机制，只要在初始化 Retrofit 对象的时候使用`addCallAdapterFactory()`，添加你自己的执行机制即可。

这个执行机制(Execution Mechanism)翻译到中文不是很合适，按我的理解，这个所谓的执行方式，就是生成的函数有个默认的返回值，但是返回值类型不一定是你需要的，所以，使用这个对象将生成函数的返回值转化成你需要的类型。

#### 由 OkHttp 提供底层技术支持

就是说引入 Retrofit2 会默认引入 OkHttp 到你的项目中。

#### 极好的效率（这个我没找到相应的测试，只是 Jake Wharton 在演讲中提到了）
