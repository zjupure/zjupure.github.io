---
layout:     post
title:      "Okhttp源码解析"
date:       2017-04-03 12:00:00
author:     "zjupure"
header-img: "img/post-bg-01.jpg"
tags:
- okhttp
- priceple
- http
- network
---

#### 前言
本文基于okhttp 3.4.1版本，主要对okhttp的特点、基本使用和底层原理进行了分析，给出了部分源码。


#### 特点

* 支持HTTP2/SPDY
* 自动维护Socket连接池，减少握手次数
* 自动选择最好的Socket路线，支持自动重连和重定向
* 队列线程池，轻松处理异步请求的并发
* 拦截器处理请求与响应（透明Gzip压缩，Logging）
* 基于Header的缓存策略；

#### 基本概念

* Request：http请求，包含url，method，headers，body（可选）
* Response：http响应，包含code，message，header，body（可选）
* Call：http的网络请求任务，由于Request/Response的重写、重定向和自动重连等，一个Call可能对应多个Request/Response
* Dispatcher：任务调度器，主要处理异步任务
* URL：http or https形式，如https://github.com/square/okhttp
* Address：用于标示一个Webserver的静态配置信息，包括协议、主机域名和端口号；共享同一个Address的URLs可以共享和复用底层的TCP Socket连接
* Route：提供连接Webserver所需的动态路由信息，包括具体的IP地址（DNS查询获取），Proxy类型（直连/Http代理），代理服务器地址，TLS版本；一个Address可能对应有多个Route，比如一个Webserver可能部署在多个数据中，DNS查询会返回多个IP地址，相应的就有多个Route可以连接

#### Connection-建立连接

1. 使用URL和OkHttpClient配置创建一个Address；
2. 尝试从Connection Pool中获取一个Address对应的可用连接；
3. 连接池中没有可用连接，选择一个Route进行尝试；该步骤通常指进行DNS查询；
4. 如果是新的Route，它就建立一个Socket直接连接、TLS隧道或TLS直接连接；
5. 发送HTTP请求并读取响应

**Note**：建立连接失败时，Okhttp会自动选择下一个Route进行重试；接收到网络响应之后，Socket连接会被放回连接池以做复用；Socket连接空闲一段时间（默认5分钟）之后会自动关闭，回收资源。

#### OkHttp请求流程

当使用OkHttpClient.newCall(reqeust)创建一个Call并调用execute()/enqueue()时，实际上只是将请求任务Call放到Dispatcher中

真正开始执行网络请求的方法在于`RealCall#getResponseWithInterceptorChain()`

```Java
private Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!retryAndFollowUpInterceptor.isForWebSocket()) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(
        retryAndFollowUpInterceptor.isForWebSocket()));

    Interceptor.Chain chain = new RealInterceptorChain(
        interceptors, null, null, null, 0, originalRequest);
    return chain.proceed(originalRequest);
  }
```

**请求流程图及拦截器**
![OkHttp请求流程]({{ site.url }}/img/okhttp/okhttp_request_seq.png)

#### Dispatcher--队列管理

OkHttp的异步调度采用了反向代理模式，Dispatcher是一个调度器，把满足条件的任务Call丢到线程池中执行
![反向代理模式]({{ site.url }}/img/okhttp/okhttp_dispatcher.png)

#### Interceptor/Chain--拦截器链

Okhttp采用了责任链模式，把Request请求每个阶段的事通过拦截器进行明确分工，所有拦截器形成了链式调用。

```Java
public interface Interceptor {
  Response intercept(Chain chain) throws IOException;

  interface Chain {
    Request request();

    Response proceed(Request request) throws IOException;

    Connection connection();
  }
}
```

Chain的实现类是`RealInterceptorChain`，入口是chain.proceed(request)-->interceptor.intercept(chain)-->chain.proceed(request)，这样直到调用的最后一个interceptor。

```
public Response proceed(Request request, StreamAllocation streamAllocation, HttpStream httpStream,
      Connection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    if (this.httpStream != null && !sameConnection(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.httpStream != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpStream, connection, index + 1, request);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpStream != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    return response;
  }
```

#### Interceptor--拦截器

Okhttp拦截器分为内置的和用户自定义的两种，本质都是实现interceptor接口。

![]({{ site.url }}/img/okhttp/okhttp_interceptors.png)

##### 内置拦截器

* RetryAndFollowUpInterceptor: 处理多Router的重试，302跳转，authorization授权登录重试等逻辑
* BridgetInterceptor：Gzip透明压缩，自动添加Host，Keep-Alive，Cookie，User-Agent等字段
* CacheInterceptor：处理本地磁盘缓存相关逻辑
* ConnectInterceptor：处理链接建立的过程
* CallServerInterceptor：发起一次http请求，并处理响应返回

##### 自定义拦截器

* Application Interceptor: 应用层拦截器
* Network Interceprot: 网络层拦截器

拦截器中可以重写Request，重写Response，追踪重定向Request，自动重试请求，Log输出等，你可以实现自己的拦截器插入到Okhttp内置的拦截器中。

拦截器中重写Response，不调用chain.proceed(request)返回，可以结束向下调用，提前终止拦截器链。

#### CacheInterceptor--缓存处理

* Cache：磁盘缓存，底层采用了DiskLruCache策略
* CacheControl：缓存控制，配置请求的缓存指令，可以指定之使用缓存，强制请求网络
* CacheStrategy：缓存策略，根据orignalRequest和cacheCandidate输出networkrequest和cahceResponse，然后决定是使用Cache还是走网络请求，实现遵循标准的HTTP语义。

networkRequest | cacheResponse | result
- | - | :-
Null | Null | only-if-cached，缓存不可用，返回504错误
Null | Non Null | 使用缓存，不请求网络
Non Null | Null | 缓存不可用，直接请求网络
Non Null | Non Null | 发送修改过的请求，网络响应返回后决定

##### DiskLRUCache--磁盘缓存实现

* LinkedHashMap实现LRU页面置换算法
* FileSystem接口，底层使用Okio对File进行封装，简化IO操作
* DiskLruCache.Eidtor，增加锁同步机制，实现了对FileSystem高级封装
* DiskLruCache.Entry，维护key对应的文件列表（.0,.1)
* DiskLruCache.SnapShot，Entry对应的快照，source列表
* 后台清理线程，会自动清理过期缓存，重建Journal文件

#### ConnectInterceptor--建立Socket连接

通过StreamAllocation类去连接池中找到一个可复用的连接或重新建立一个新的连接；

StreamAllocation.newStream()该方法获取一个可用连接，并把输入输出流(source/sink)封装到HttpStream中。


```Java
public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpStream httpStream = streamAllocation.newStream(client, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpStream, connection);
  }
```

##### StreamAllocation--管理Socket连接

该类负责处理Socket连接的细节，绑定一个Address，选择合适的路线并处理自动重连，通过Connect Pool实现连接的复用，把RealConnection的流封装成HTTPStream供上层访问；Http请求介绍，能够自动释放Socket链路

![socket_build]({{ site.url }}/img/okhttp/okhttp_socket_build.png)

##### RouterSelector--路由选择

与Address进行绑定，根据IP地址和Proxy类型，生成可用的Route列表，通过next()方法选择当前的Route。

RouteDatabase维护失败的Route黑名单，下次相同host的请求时自动回避黑名单中的路由，放到最后重试。

```Java
public Route next() throws IOException {
    // Compute the next route to attempt.
    if (!hasNextInetSocketAddress()) {
      if (!hasNextProxy()) {
        if (!hasNextPostponed()) {
          throw new NoSuchElementException();
        }
        return nextPostponed();
      }
      lastProxy = nextProxy();
    }
    lastInetSocketAddress = nextInetSocketAddress();

    Route route = new Route(address, lastProxy, lastInetSocketAddress);
    if (routeDatabase.shouldPostpone(route)) {
      postponedRoutes.add(route);
      // We will only recurse in order to skip previously failed routes. They will be tried last.
      return next();
    }

    return route;
  }
```

##### RealConnection--建立连接

* 维护TCP Socket和SSL Socket，维护输入输出流（Source/Sink）
* 使用connect()方法建立连接
* Http请求只需要创建TCP Socket连接；Https请求在TCP Socket基础上，会创建SSL Socket连接，会进行证书校验和握手。

![connection_build]({{ site.url }}/img/okhttp/okhttp_connection_build.png)

##### ConnectionPool--连接池复用

* 相同Address的请求可以共享同一个Connection
* 默认5个空闲的Keep-Alive连接用于复用，存活时间5分钟
* 后台回收线程会定期清理过期的连接，释放资源

**资源释放**
1. 引用计数算法：RealConnection维护一份StreamAllocation的弱引用列表，通过StreamAllocation的acquire()和release()进行引用计数
2. 标记清除算法：找出最不活跃的空闲连接/泄露连接，根据空闲时间决定立即执行清理操作，还是wait一段时间后再清理

#### CallServerInterceptor--执行网络请求

HTTPStream封装了输入输出流（Source/Sink），直接通过HttpStream进行数据的读写操作。

```Java
public Response intercept(Chain chain) throws IOException {
    HttpStream httpStream = ((RealInterceptorChain) chain).httpStream();
    StreamAllocation streamAllocation = ((RealInterceptorChain) chain).streamAllocation();
    Request request = chain.request();

    long sentRequestMillis = System.currentTimeMillis();
    httpStream.writeRequestHeaders(request);

    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      Sink requestBodyOut = httpStream.createRequestBody(request, request.body().contentLength());
      BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
      request.body().writeTo(bufferedRequestBody);
      bufferedRequestBody.close();
    }

    httpStream.finishRequest();

    Response response = httpStream.readResponseHeaders()
        .request(request)
        .handshake(streamAllocation.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    if (!forWebSocket || response.code() != 101) {
      response = response.newBuilder()
          .body(httpStream.openResponseBody(response))
          .build();
    }

    if ("close".equalsIgnoreCase(response.request().header("Connection"))
        || "close".equalsIgnoreCase(response.header("Connection"))) {
      streamAllocation.noNewStreams();
    }

    int code = response.code();
    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
      throw new ProtocolException(
          "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
    }

    return response;
  }
```

#### 结语

以上就是Okhttp涉及的核心原理，没有贴太多的代码，主要从整体架构和请求流程上对Okhttp进行了梳理。









