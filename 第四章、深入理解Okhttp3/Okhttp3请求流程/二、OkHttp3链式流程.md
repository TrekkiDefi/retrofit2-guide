#### 请求流程
首先我们先看一看它的请求流程，在Okhttp3中请求是基于拦截器原理。

>源码参考：okhttp3.RealCall.java

```java
Response getResponseWithInterceptorChain() throws IOException {
    List<Interceptor> interceptors = new ArrayList();
    interceptors.addAll(this.client.interceptors());
    interceptors.add(this.retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(this.client.cookieJar()));
    interceptors.add(new CacheInterceptor(this.client.internalCache()));
    interceptors.add(new ConnectInterceptor(this.client));
    if(!this.forWebSocket) {
        interceptors.addAll(this.client.networkInterceptors());
    }

    interceptors.add(new CallServerInterceptor(this.forWebSocket));
    Chain chain = new RealInterceptorChain(interceptors, (StreamAllocation)null, (HttpCodec)null, (Connection)null, 0, this.originalRequest);
    return chain.proceed(this.originalRequest);
}
```
研究Okhttp3的源码，从此处开始，一个一个拦截器，读懂即可。不得不说这种方式真的很棒，清晰明了。

![](/assets/okhttp3_chain_process.jpg)

通过上图，想必对Okhttp3的实现方式，已经有了基本的认识下面我们就一步一具体分析。

##### 构造OkHttpClient

OkHttpClient采用了建造者设计模式来实例化。本身有多个字段用于全局对象，比去Cache、Dns等

- 静态代码块构造全局缓存
    ```java
    //TODO 构造全局缓存
    static {
        Internal.instance = new Internal() {
          ...
          @Override public void setCache(OkHttpClient.Builder builder, InternalCache internalCache) {
            // 写入缓存，指的是响应数据缓存
            builder.setInternalCache(internalCache);
          }
          ...
          @Override public RealConnection get(
              ConnectionPool pool, Address address, StreamAllocation streamAllocation) {
            // 从缓存中获取有效的连接，仅支持Http/2，本质就是从内存的ConnectionPool中的Deque读取
            return pool.get(address, streamAllocation);
          }
          ...
          @Override public void put(ConnectionPool pool, RealConnection connection) {
            // 将连接缓存到连接池中
            pool.put(connection);
          }
          @Override public RouteDatabase routeDatabase(ConnectionPool connectionPool) {
            // 线路的缓存，多host多ip的情况
            return connectionPool.routeDatabase;
          }
          ...
          // 此处有很多种数据缓存处理，不了解并不影响代码分析，如有兴趣可自行研究
        };
      }
    ```
- Dispatcher， 分发请求，内部是有一个ThreadPoolExecutor
- Proxy， 代理连接，分三种类型直接（DIRECT）、HTTP、SOCKS。
- ProxySelector，线路选择器，对应Okhttp的一大特点，自行线路选择，找到合适的连接
- Cache， 真正的缓存实现
- SSLSocketFactory， Https的支持
- ConnectionPool， 连接池
- Dns，dns解析（Java实现）
- 其他，如超时时间等

##### 发起请求
我们都知道一个请求有多部分组成，同样Okhttp3创建一个请求也要多部分。

- 构造请求头
- 构造请求体
- 构造Request

通过以上三步我们就可以完成一次请求。

###### 构造请求头
Header并非每个请求都需要，要看与服务端是如何定义的，通常一个请求会默认一些头，比如Content-Type，Accept-Encoding，Connection等对应http协议

Header本质上就是一个Map，只是在封装了一层而已，但是Okhttp3的实现不是这样，而是一个**String数组**，key + value的形式，即一个头占用数组的两位：

![](/assets/header_array.jpg)

每一个色块对应一个header，代码如下：

>源码参考：okhttp3.Headers.java

```java
public final class Headers {
  private final String[] namesAndValues;

  Headers(Builder builder) {
    this.namesAndValues = builder.namesAndValues.toArray(new String[builder.namesAndValues.size()]);
  }

  private Headers(String[] namesAndValues) {
    this.namesAndValues = namesAndValues;
  }
  
  ...
  
  public static final class Builder {
      final List<String> namesAndValues = new ArrayList<>(20);
      Builder addLenient(String name, String value) {
        namesAndValues.add(name);
        namesAndValues.add(value.trim());
        return this;
      }
      ...
  }
}
```
###### 构造请求体
请求体有多种形式，对应的父类是RequestBody，有文件形式、Json等，MediaType决定了是何种形式，通常我们用的是FromBody和MultipartBody

**1、FromBody**

>源码参考：okhttp3.FromBody.java

```java
package okhttp3;

public final class FormBody extends RequestBody {
  private static final MediaType CONTENT_TYPE =
      MediaType.parse("application/x-www-form-urlencoded");

  private final List<String> encodedNames;// 参数名称
  private final List<String> encodedValues;// 参数值

  FormBody(List<String> encodedNames, List<String> encodedValues) {
    this.encodedNames = Util.immutableList(encodedNames);
    this.encodedValues = Util.immutableList(encodedValues);
  }
  ...
  public static final class Builder {
    private final List<String> names = new ArrayList<>();
    private final List<String> values = new ArrayList<>();

    public Builder add(String name, String value) {
      names.add(HttpUrl.canonicalize(name, FORM_ENCODE_SET, false, false, true, true));
      values.add(HttpUrl.canonicalize(value, FORM_ENCODE_SET, false, false, true, true));
      return this;
    }

    public Builder addEncoded(String name, String value) {
      names.add(HttpUrl.canonicalize(name, FORM_ENCODE_SET, true, false, true, true));
      values.add(HttpUrl.canonicalize(value, FORM_ENCODE_SET, true, false, true, true));
      return this;
    }

    public FormBody build() {
      return new FormBody(names, values);
    }
  }
}
```
**2、MultipartBody**
MultipartBody原理基本一致，区别在于他可以发送表单的同时也可以发送文件数据，再次不在赘述。

###### 构造Request
有了上面两个步骤，接下了就自然而然产生一个Request，顾名思义它就是对请求的封装，包括请求方式，请求头，请求体，请求路径等,源代码码也是比较简单，一看即明白。

>源码参考：okhttp3.Request.java

```java
public final class Request {
  final HttpUrl url;
  final String method;
  final Headers headers;
  final RequestBody body;
  final Object tag;
  
  ...
```

#### Call
##### 调用链
Call是一个接口，实现类只有一个RealCall，上面我们提到的流程就是在RealCall中。

>源码参考：okhttp3.Call.java

```java
public interface Call extends Cloneable {
  Request request(); // 请求封装的数据
  Response execute() throws IOException; // 同步执行
  void enqueue(Callback responseCallback); // 异步执行
  void cancel(); // 取消请求
  boolean isExecuted();
  boolean isCanceled();
  Call clone();
  interface Factory {
    Call newCall(Request request);
  }
}
```

>源码参考：okhttp3.RealCall.java

```java
package okhttp3;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import okhttp3.Interceptor.Chain;
import okhttp3.internal.NamedRunnable;
import okhttp3.internal.cache.CacheInterceptor;
import okhttp3.internal.connection.ConnectInterceptor;
import okhttp3.internal.connection.StreamAllocation;
import okhttp3.internal.http.BridgeInterceptor;
import okhttp3.internal.http.CallServerInterceptor;
import okhttp3.internal.http.HttpCodec;
import okhttp3.internal.http.RealInterceptorChain;
import okhttp3.internal.http.RetryAndFollowUpInterceptor;
import okhttp3.internal.platform.Platform;

final class RealCall implements Call {
    final OkHttpClient client;//OkHttpClient
    final RetryAndFollowUpInterceptor retryAndFollowUpInterceptor;// 重试拦截器
    final Request originalRequest;//发起此Call的原始请求
    final boolean forWebSocket;
    private boolean executed;

    RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
        this.client = client;
        this.originalRequest = originalRequest;
        this.forWebSocket = forWebSocket;
        this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
    }

    /**
    * 返回发起此Call的原始请求
    */
    public Request request() {
        return this.originalRequest;
    }
    /**
    * 立即调用请求，并阻塞，直到响应可以被处理或出现错误。
    *   
    *   为了避免资源泄漏，调用者(callers)应该关闭{@link Response}，这反过来将关闭底层的{@link ResponseBody}。
    *   
    *   <pre>@{code 
    *       //确保响应（和底层响应体）关闭 
    *       try (Response response = client.newCall(request).execute()) {
    *           ...
    *   }}</pre>
    *   
    *   调用者可以使用响应的{@link Response#body}方法读取响应主体。
    *   
    *   为了避免资源泄露，调用者必须{@linkplain ResponseBody 关闭响应体}或响应。
    *   
    *   请注意，传输层成功（接收HTTP响应码，响应头和正文）并不一定表示应用层成功：响应可能仍然表示不乐观的HTTP响应代码，如404或500。
    */
    // 同步执行的实现
    public Response execute() throws IOException {
        // 一个call只能执行一次
        synchronized (this) {//同步代码块，当前Call对象作为锁对象
            //如果已经执行，抛出异常，否则设置执行状态为已执行
            if (executed) throw new IllegalStateException("Already Executed");
            executed = true;
        }
        captureCallStackTrace();
        try {
            client.dispatcher().executed(this);
            // 获取结果，即执行多个链接器的调用链
            Response result = getResponseWithInterceptorChain();
            if (result == null) throw new IOException("Canceled");
            return result;
        } finally {
            client.dispatcher().finished(this);
        }
    }

    private void captureCallStackTrace() {
        Object callStackTrace = Platform.get().getStackTraceForCloseable("response.body().close()");
        this.retryAndFollowUpInterceptor.setCallStackTrace(callStackTrace);
    }

    //  异步执行
    public void enqueue(Callback responseCallback) {
        synchronized(this) {
            if(this.executed) {
                throw new IllegalStateException("Already Executed");
            }

            this.executed = true;
        }

        this.captureCallStackTrace();
        // AsyncCall是一个Runnable的实现类，同时一个是RealCall的内部类
        this.client.dispatcher().enqueue(new RealCall.AsyncCall(responseCallback));
    }

    public void cancel() {
        this.retryAndFollowUpInterceptor.cancel();
    }

    public synchronized boolean isExecuted() {
        return this.executed;
    }

    public boolean isCanceled() {
        return this.retryAndFollowUpInterceptor.isCanceled();
    }

    public RealCall clone() {
        return new RealCall(this.client, this.originalRequest, this.forWebSocket);
    }

    StreamAllocation streamAllocation() {
        return this.retryAndFollowUpInterceptor.streamAllocation();
    }

    String toLoggableString() {
        return (this.isCanceled()?"canceled ":"") + (this.forWebSocket?"web socket":"call") + " to " + this.redactedUrl();
    }

    String redactedUrl() {
        return this.originalRequest.url().redact();
    }

    Response getResponseWithInterceptorChain() throws IOException {
        // Build a full stack of interceptors.
        // 拦截器栈
        List<Interceptor> interceptors = new ArrayList<>();
        // 前文说过的 自定义拦截器
        interceptors.addAll(client.interceptors());
        // 重试拦截器，网络错误、请求失败、重定向等
        interceptors.add(retryAndFollowUpInterceptor);
        // 桥接拦截器，主要是重构请求头即Header
        interceptors.add(new BridgeInterceptor(client.cookieJar()));
        // 缓存拦截器
        interceptors.add(new CacheInterceptor(client.internalCache()));
        // 连接拦截器，连接服务器，https包装
        interceptors.add(new ConnectInterceptor(client));
        // 网络拦截器，websockt不支持，同样是自定义拦截器
        if (!forWebSocket) {
          interceptors.addAll(client.networkInterceptors());
        }
        // 服务拦截器，主要是发送（write、input）、读取（read、output）数据
        interceptors.add(new CallServerInterceptor(forWebSocket));
        // 开启调用链
        Interceptor.Chain chain = new RealInterceptorChain(
            interceptors, null, null, null, 0, originalRequest);
        return chain.proceed(originalRequest);
    }

    // 异步执行的线程封装
    final class AsyncCall extends NamedRunnable {
        private final Callback responseCallback;

        AsyncCall(Callback responseCallback) {
            super("OkHttp %s", new Object[]{RealCall.this.redactedUrl()});
            this.responseCallback = responseCallback;
        }

        String host() {
            return RealCall.this.originalRequest.url().host();
        }

        Request request() {
            return RealCall.this.originalRequest;
        }

        RealCall get() {
            return RealCall.this;
        }

        protected void execute() {
            boolean signalledCallback = false;

            try {
                // 执行调用链
                Response response = RealCall.this.getResponseWithInterceptorChain();
                if(RealCall.this.retryAndFollowUpInterceptor.isCanceled()) {
                    signalledCallback = true;
                    this.responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
                } else {
                    signalledCallback = true;
                    this.responseCallback.onResponse(RealCall.this, response);
                }
            } catch (IOException var6) {
                if(signalledCallback) {
                    Platform.get().log(4, "Callback failure for " + RealCall.this.toLoggableString(), var6);
                } else {
                    this.responseCallback.onFailure(RealCall.this, var6);
                }
            } finally {
                RealCall.this.client.dispatcher().finished(this);
            }

        }
    }
}
```
看来上面的代码，是不是被人家的想法折服。更精妙的还在后面呢，不要急。到此，一个请求已经完成，发送和接收，（不关心内部实现的话）。接下来我们再来看看，拦截器是如何工作的。

拦截器工作原理：
```text
拦截器，就像水管一样，把一节一节的水管（拦截器）串起来，形成一个回路，从而能够把数据发送到服务器，又能接受返回的数据，而每一节水管（拦截器）都有自己的作用，分别处理不同东西，比如净化水、过滤杂质等。
Interceptor是一个接口，主要是对请求和响应的处理，而实现拦截器调用链的是其内部接口Chain。
```
Interceptor.Chain的实现类只有一个RealInterceptorChain，也是处理调用过程的实现，其内部有个List装载了所有拦截器，

大家想必也猜到了，对，就是迭代，当然也不是简简单单的迭代了事。让我们看看源码实现。

```java
Response getResponseWithInterceptorChain() throws IOException {
    ...
    // 开启调用链
    Interceptor.Chain chain = new RealInterceptorChain(
        interceptors, null, null, null, 0, originalRequest);
    return chain.proceed(originalRequest);
}
```
拦截器调用链的最开始只传入参数`List<Interceptor> interceptors`、`Request request`、`index`，并且index传入0。

>源码参考：okhttp3.internal.http.RealInterceptorChain.java

```java
package okhttp3.internal.http;

import java.io.IOException;
import java.util.List;
import okhttp3.Connection;
import okhttp3.HttpUrl;
import okhttp3.Interceptor;
import okhttp3.Request;
import okhttp3.Response;
import okhttp3.internal.connection.StreamAllocation;

/**
 * 一个具体的拦截链，承载整个拦截链：所有应用拦截器，OkHttp核心，所有网络拦截器，最后是网络调用者。
 */
// RealCall.getResponseWithInterceptorChain()中创建了一个实例
public final class RealInterceptorChain implements Interceptor.Chain {
  // RealCall.getResponseWithInterceptorChain()中创建实例时通过构造器赋值
  private final List<Interceptor> interceptors;//所有拦截器
  private final Request request;
  
  // 下面属性会在执行拦截器的过程中一步一步赋值
  private final StreamAllocation streamAllocation;
  private final HttpCodec httpCodec;
  private final Connection connection;
  
  private final int index;
  private int calls;

  public RealInterceptorChain(List<Interceptor> interceptors, StreamAllocation streamAllocation,
      HttpCodec httpCodec, Connection connection, int index, Request request) {
    this.interceptors = interceptors;
    this.connection = connection;
    this.streamAllocation = streamAllocation;
    this.httpCodec = httpCodec;
    this.index = index;
    this.request = request;
  }

  @Override public Connection connection() {
    return connection;
  }

  public StreamAllocation streamAllocation() {
    return streamAllocation;
  }

  public HttpCodec httpStream() {
    return httpCodec;
  }

  @Override public Request request() {
    return request;
  }

  // 实现了父类proceed方法
  @Override public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
  }

  // 处理调用
  public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      Connection connection) throws IOException {
    // 当前要执行的拦截器的索引，[0, interceptors.size())
    if (index >= interceptors.size()) throw new AssertionError();
    // 标记已调用
    calls++;

    /**
    * 注：后面我们会了解到，ConnectInterceptor拦截器:
    * 创建网络读写流必要的HttpCodec，用于请求编码和网络响应解码处理。
    * 复用或建立Socket连接RealConnection，用于网络数据传输。
    * 网络数据流具体处理细节交给CallServerInterceptor节点。
    * 
    * 如果this.httpCodec不为空，则当前要执行的拦截器为CallServerInterceptor。
    */
    // If we already have a stream, confirm that the incoming request will use it.
    // 即保证传入的请求与当前持有的Connection的主机和端口号相同。
    if (this.httpCodec != null && !sameConnection(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    // 即保证调用只会执行一次。必须且只能调用一次Chain.proceed()方法，保证链式调用唯一。
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.
    // 创建新的RealInterceptorChain实例，这里一个拦截器对应一个RealInterceptorChain实例。
    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpCodec, connection, index + 1, request);
    // 取出下一个拦截器
    Interceptor interceptor = interceptors.get(index);
    // 执行拦截器方法，拦截器中又会调用proceed()方法
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    return response;
  }

  private boolean sameConnection(HttpUrl url) {
    return url.host().equals(connection.route().address().url().host())
        && url.port() == connection.route().address().url().port();
  }
}
```
怎么才能够做到迭代执行呢，主要是下面几行代码：
```java
RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpCodec, connection, index + 1, request);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
```
这里我们知道了，一个拦截器实例对应一个RealInterceptorChain实例。

在`Response response = interceptor.intercept(next);`里又执行了下面的逻辑（我们以BridgeInterceptor拦截器举例）：

>源码参考：okhttp3.internal.http.BridgeInterceptor.java

```java
package okhttp3.internal.http;

import java.io.IOException;
import java.util.List;
import okhttp3.Cookie;
import okhttp3.CookieJar;
import okhttp3.Headers;
import okhttp3.Interceptor;
import okhttp3.MediaType;
import okhttp3.Request;
import okhttp3.RequestBody;
import okhttp3.Response;
import okhttp3.internal.Version;
import okio.GzipSource;
import okio.Okio;

import static okhttp3.internal.Util.hostHeader;

public final class BridgeInterceptor implements Interceptor {
  private final CookieJar cookieJar;

  public BridgeInterceptor(CookieJar cookieJar) {
    this.cookieJar = cookieJar;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    ...
    // 构造Request，并继续执行chain.proceed()方法。
    Response networkResponse = chain.proceed(requestBuilder.build());

    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);
    ...

    return responseBuilder.build();
  }
  ...
}
```

###### 总结
看了上面的代码是不是还不明白，到底怎么实现的，实际上就是迭代+递归。

每一个RealInterceptorChain会对应一个Interceptor，然后Interceptor在产生下一个RealInterceptorChain，直到List迭代完成。

Okhttp3的调用流程基本原理就是这样，重要的是思想，整个流程一气呵成，完全解耦。

![](/assets/RealInterceptorChain.jpg)

###### 扩展
最后我们再谈一下调用的结束流程。

**1、取消请求**
通过以下方式创建Call对象：

>源码参考：okhttp3.OkHttpClient.java

```java
package okhttp3;

public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory {
  ...
  static {
    Internal.instance = new Internal() {
      ...
      @Override public Call newWebSocketCall(OkHttpClient client, Request originalRequest) {
        return new RealCall(client, originalRequest, true);
      }
    };
  }
  
  ...
  
  /**
   * Prepares the {@code request} to be executed at some point in the future.
   */
  @Override public Call newCall(Request request) {
    return new RealCall(this, request, false /* for web socket */);
  }
}
```

>源码参考：okhttp3.RealCall.java

```java
final class RealCall implements Call {
    RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
        this.client = client;
        this.originalRequest = originalRequest;
        this.forWebSocket = forWebSocket;
        this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
    }
    
    ...
    
    @Override public void cancel() {
        retryAndFollowUpInterceptor.cancel();
    }
}
```

>源码参考：okhttp3.internal.http.RetryAndFollowUpInterceptor.java

```java
package okhttp3.internal.http;
public final class RetryAndFollowUpInterceptor implements Interceptor {
    ...
    /**
    * 如果套接字(socket)当前保持连接，则立即关闭连接。
    * 使用它来中断来自任何线程的"in-flight"的请求。
    * 调用者有责任关闭请求体和响应体流; 否则资源可能会泄漏。
    * 该方法是可以同时调用的，但提供有限的保证。 
    * 如果已经建立了传输层连接（例如HTTP/2流），则会终止。
    * 如果正在建立套接字连接，则会终止。
    */
    public void cancel() {
        canceled = true;
        StreamAllocation streamAllocation = this.streamAllocation;
        if (streamAllocation != null) streamAllocation.cancel();
      }
    ...
}
```

>源码参考：okhttp3.internal.connection.StreamAllocation.java

```java
public final class StreamAllocation {//TODO StreamAllocation
    ...
    public void cancel() {
        HttpCodec codecToCancel;
        RealConnection connectionToCancel;
        synchronized (connectionPool) {
            canceled = true;
            codecToCancel = codec;
            connectionToCancel = connection;
        }
    if (codecToCancel != null) {
        codecToCancel.cancel();
    } else if (connectionToCancel != null) {
        connectionToCancel.cancel();
    }
    }
    ...
}
```
Call对象创建会默认创建拦截器RetryAndFollowUpInterceptor。

调用取消，实际上是将拦截器RetryAndFollowUpInterceptor的**canceled**状态置为**true**，并关闭连接。

**2、结束请求**
通过上面的学习我们知道，无论是同步还是异步，调用结束后都会执行以下代码：
```java
finally {
    client.dispatcher().finished(this);
}
```

>源码参考：okhttp3.Dispatcher.java

```java
public final class Dispatcher {
    ...
    /**
    * 由{@code AsyncCall#run}调用，用来表示完成
    * @param call 异步调用Call
    */
    void finished(AsyncCall call) {
        finished(runningAsyncCalls, call, true);
    }
    
    /**
    * 由{@code Call#execute}调用，用来表示完成
    * @param call 同步调用Call
    */
    void finished(RealCall call) {
        finished(runningSyncCalls, call, false);
    }
    
    /**
    * @param calls 同步队列/异步队列
    * @param call 当前调用
    * @param promoteCalls 标记是否是异步调用
    */
    private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
        int runningCallsCount;//正在运行的同步调用和异步调用的总数
        Runnable idleCallback;//空闲线程
        synchronized (this) {//同步代码块，当前Dispatcher对象作为锁对象
            //将当前调用从队列移除，如果队列中不存在，抛出异常
            if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
            //如果是异步调用，则执行promoteCalls()方法逻辑，处理Ready异步队列中的调用
            if (promoteCalls) promoteCalls();
            runningCallsCount = runningCallsCount();//正在运行的同步调用和异步调用的总数
            idleCallback = this.idleCallback;
        }
        // 当前正在运行的调用为0时，运行一个idleCallback线程。
        if (runningCallsCount == 0 && idleCallback != null) {
            idleCallback.run();
        }
    }
    
    ...
    /**
    * 异步调用执行逻辑
    * 
    * 下面异步调用我们提及到enqueue()方法:<br>
    *       如果正在运行的调用数小于maxRequests，并且正在运行的请求同一个主机的调用数小于maxRequestsPerHost，则放入Running异步队列，并放入线程池执行。<br>
    *       否则只是放入Ready异步队列（readyAsyncCalls）等待执行。<br>
    * 对于放入Ready异步队列的调用什么时机放入Running异步队列，并放入线程池运行呢？看下面的逻辑。
    */
    private void promoteCalls() {
        // 如果Running异步队列已经运行最大容量，则不做任何处理，直接返回。
        if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.
        // 如果Ready异步队列为空，则没有待运行的调用，直接返回。
        if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.
        
        // Ready异步队列有调用，则将其放入Running异步队列，并放入线程池执行
        for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
            AsyncCall call = i.next();
            // 这里还有一个限制，就是正在运行的请求中，正在运行的请求同一个主机的调用数小于maxRequestsPerHost
            if (runningCallsForHost(call) < maxRequestsPerHost) {
                i.remove();
                runningAsyncCalls.add(call);
                executorService().execute(call);
            }
            // 如果Running异步队列已经运行最大容量，则不再添加Ready异步队列中的调用
            if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
        }
    }
    
    /**
    * 返回与{@code call}共享一个主机的正在运行的调用数
    */
    private int runningCallsForHost(AsyncCall call) {
        int result = 0;
        for (AsyncCall c : runningAsyncCalls) {
            if (c.host().equals(call.host())) result++;
        }
        return result;
    }
}
```

##### 同步执行
Next，我们单独分析下**execute**方法：
```java
public Response execute() throws IOException {
    synchronized (this) {
        if (executed) throw new IllegalStateException("Already Executed");
        executed = true;
    }
    captureCallStackTrace();
    try {
        client.dispatcher().executed(this);
        Response result = getResponseWithInterceptorChain();
        if (result == null) throw new IOException("Canceled");
        return result;
    } finally {
        client.dispatcher().finished(this);
    }
}
```
**首先，`client.dispatcher().executed(this);`方法的源代码如下：**
```java
public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory {
    ...
    // 获取Dispatcher
    public Dispatcher dispatcher() {
        return dispatcher;
    }
    ...
}
```
```java
/**
* 执行异步请求时的策略。
* 
* 每个调度程序使用{@link ExecutorService}在内部运行调用。 如果您提供自己的执行程序，它应该能够并发运行{@linkplain #getMaxRequests 配置的maximum}数量的调用。
*/
public final class Dispatcher {
    
    /**
    * 运行同步调用，包括尚未完成的取消调用。
    */
    private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
    
    ...
    
    /**
    * 由{@code Call#execute}调用，标记为"in-flight"
    */
    synchronized void executed(RealCall call) {
        runningSyncCalls.add(call);
    }
}
```
这里我们对ArrayDeque做一个简单的介绍：

ArrayDeque是{@link Deque}接口的可调整大小的数组实现。ArrayDeque没有容量限制; 它们根据需要增长以支持使用。

它们不是线程安全的; 在没有额外同步的情况下，它们不支持多线程的并发访问。 

不支持Null元素。

当用作堆栈时，此类比{@link Stack}更快，并且当用作队列时比{@link LinkedList}更快。

**接着，执行`Response result = getResponseWithInterceptorChain()`;**

`getResponseWithInterceptorChain`方法源码如下：
```java
// 开始执行整个请求
Response getResponseWithInterceptorChain() throws IOException {
    List<Interceptor> interceptors = new ArrayList();
    interceptors.addAll(this.client.interceptors());
    interceptors.add(this.retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(this.client.cookieJar()));
    interceptors.add(new CacheInterceptor(this.client.internalCache()));
    interceptors.add(new ConnectInterceptor(this.client));
    if(!this.forWebSocket) {
        interceptors.addAll(this.client.networkInterceptors());
    }

    interceptors.add(new CallServerInterceptor(this.forWebSocket));
    Chain chain = new RealInterceptorChain(interceptors, (StreamAllocation)null, (HttpCodec)null, (Connection)null, 0, this.originalRequest);
    return chain.proceed(this.originalRequest);
}
```

##### 异步执行
Next，我们单独分析下**enqueue**方法：
```java
public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```
**首先，`client.dispatcher().enqueue(new AsyncCall(responseCallback));`方法的源代码如下：**
```java
public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory {
    ...
    // 获取Dispatcher
    public Dispatcher dispatcher() {
        return dispatcher;
    }
    ...
}
```
```java
public final class Dispatcher {
    private int maxRequests = 64;
    
    // 运行异步调用，包括尚未完成的取消调用。
    private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();
    
    // 按照他们将要运行的顺序进行准备就绪的异步调用。
    private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();
    ...
    
    synchronized void enqueue(AsyncCall call) {
        // 如果正在运行的调用数(Running异步队列的大小)小于maxRequests，并且正在运行的请求同一个主机的调用数小于maxRequestsPerHost，则放入Running异步队列，并放入线程池执行。
        // 否则只是放入Ready异步队列（readyAsyncCalls）等待执行。
        // 线程池运行调用线程，进而调用getResponseWithInterceptorChain()方法执行调用链。
        if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
          runningAsyncCalls.add(call);
          executorService().execute(call);
        } else {
          readyAsyncCalls.add(call);
        }
    }
    
    /**
    * 返回与{@code call}共享一个主机的正在运行的调用数
    */
    private int runningCallsForHost(AsyncCall call) {
        int result = 0;
        for (AsyncCall c : runningAsyncCalls) {
          if (c.host().equals(call.host())) result++;
        }
        return result;
    }
}
```

>源码参考：okhttp3.internal.NamedRunnable.java

```java
public abstract class NamedRunnable implements Runnable {
    protected final String name;
    
    public NamedRunnable(String format, Object... args) {
        this.name = Util.format(format, args);
    }
    
    @Override public final void run() {
        String oldName = Thread.currentThread().getName();
        Thread.currentThread().setName(name);
        try {
            execute();
        } finally {
            Thread.currentThread().setName(oldName);
        }
    }
    
    protected abstract void execute();
    }
```

>源码参考：okhttp3.RealCall.AsyncCall.java

```java
final class RealCall implements Call {
    ...
    final class AsyncCall extends NamedRunnable {
        ...
        protected void execute() {
              boolean signalledCallback = false;
              try {
                Response response = getResponseWithInterceptorChain();
                if (retryAndFollowUpInterceptor.isCanceled()) {
                  signalledCallback = true;
                  responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
                } else {
                  signalledCallback = true;
                  responseCallback.onResponse(RealCall.this, response);
                }
              } catch (IOException e) {
                if (signalledCallback) {
                  // Do not signal the callback twice!
                  Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
                } else {
                  responseCallback.onFailure(RealCall.this, e);
                }
              } finally {
                client.dispatcher().finished(this);
              }
            }
    }
}
```
由上面代码分析可知，AsyncCall 实现 NamedRunnable抽象类，而NamedRunnable 实现 Runnable接口，通过线程池执行请求。
NamedRunnable类中run()方法会调用其实现类AsyncCall的execute()方法，该方法进而调用getResponseWithInterceptorChain()方法从而执行OkHttp3链式流程。

`getResponseWithInterceptorChain`方法源码如下：
```java
// 开始执行整个请求
Response getResponseWithInterceptorChain() throws IOException {
    List<Interceptor> interceptors = new ArrayList();
    interceptors.addAll(this.client.interceptors());
    interceptors.add(this.retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(this.client.cookieJar()));
    interceptors.add(new CacheInterceptor(this.client.internalCache()));
    interceptors.add(new ConnectInterceptor(this.client));
    if(!this.forWebSocket) {
        interceptors.addAll(this.client.networkInterceptors());
    }

    interceptors.add(new CallServerInterceptor(this.forWebSocket));
    Chain chain = new RealInterceptorChain(interceptors, (StreamAllocation)null, (HttpCodec)null, (Connection)null, 0, this.originalRequest);
    return chain.proceed(this.originalRequest);
}
```

#### RetryAndFollowUpInterceptor

`RetryAndFollowUpInterceptor`用于尝试恢复失败和重定向的请求，最多支持跟踪20次重定向。

创建streamAllocation维护请求的Connections、Streams、Calls，类似中介者模式，之后交给BridgeInterceptor节点处理请求。

```java
@Override 
public Response intercept(Chain chain) throws IOException {
    ...
    // 创建用于协调Connections、Streams、Call三者关系的streamAllocation
    streamAllocation = new StreamAllocation(
        client.connectionPool(), createAddress(request.url()), callStackTrace);
    // 重定向次数
    int followUpCount = 0;
    Response priorResponse = null;
    while (true) {
      // 无限循环
      ...
      Response response = null;
      boolean releaseConnection = true;
      try {
        response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (RouteException e) {
        // 连接路由失败，请求未发送
        ...
        releaseConnection = false;
        continue;
      } catch (IOException e) {
        // 与服务端通信失败，请求已发送
        ...
        releaseConnection = false;
        continue;
      } finally {
        // 释放资源
        if (releaseConnection) {
          ...
        }
      }
      // 记录上一次的响应，一般出现在重定向情况时。
      ...
      // 判断是否是重定向的响应
      Request followUp = followUpRequest(response);
      ...
      if (followUp == null) {
        ...
        // 正常响应直接返回
        return response;
      }
      ...
      // 检查是否能够继续重定向操作
      ...
      request = followUp;
      priorResponse = response;
    }
  }
```
 
`RetryAndFollowUpInterceptor`处理下层链中节点返回的响应和抛出的异常。 依据返回的响应或抛出的异常，进行检查和恢复操作

- 关闭已建立的Socket连接
- OkHttpClient是否关闭重连，默认开启重连
- 请求是否已发送并且请求体不可重读，不可重连
- 出现的致命的异常：请求协议异常、证书验证异常等
- 是否有下一跳可尝试的路由。

```java
private boolean recover(IOException e, boolean requestSendStarted, Request userRequest) {
    // 关闭Socket
    streamAllocation.streamFailed(e);
    // 如果Application层禁止重连，则直接失败
    if (!client.retryOnConnectionFailure()) return false;
    // 是否可以再次发送请求
    if (requestSendStarted && userRequest.body() instanceof UnrepeatableRequestBody) return false;
    // 致命异常则不可恢复
    if (!isRecoverable(e, requestSendStarted)) return false;
    // 没有可以再次尝试的路由
    if (!streamAllocation.hasMoreRoutes()) return false;
    return true;
  }
```
#### BridgeInterceptor
`BridgeInterceptor`是应用层与网络层节点的桥接，补全应用层请求的头部信息，调用之后网络与缓存数据处理，最后将响应返回给上层。

```java
@Override 
public Response intercept(Chain chain) throws IOException {
    // 重写请求头部，填充必要的头部信息
    ...
    // 添加 "Accept-Encoding: gzip" header ，可以压缩请求数据
    ...
     requestBuilder.header("Accept-Encoding", "gzip");
    ...

    // 配置Cookie和代理信息
    ...
    requestBuilder.header("Cookie", cookieHeader(cookies));
    ...
    requestBuilder.header("User-Agent", Version.userAgent());
    ...
    // 把新构建的请求向下传递处理
    Response networkResponse = chain.proceed(requestBuilder.build());
    // 处理下层节点返回的响应，响应可能是缓存或者网络数据
    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);
    // 响应数据解压
    ...
    return responseBuilder.build();
  }
```
#### CacheInterceptor
- 读取本地缓存，根据请求缓存策略构建网络请求和缓存响应。
- 按照请求缓存策略，返回缓存或传递给ConnectInterceptor执行下一步数据操作。
- 处理返回的网络响应数据的缓存操作。

```java
@Override 
public Response intercept(Chain chain) throws IOException {
    // 读取本地磁盘缓存
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;
    ...
    
    // 缓存策略
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;
    
    // 如果缓存未命中，则舍弃缓存
    ...
    
    // 禁止网络请求且不存在缓存，返回504，请求失败
    if (networkRequest == null && cacheResponse == null) {
      ...
      return ...;
    }
    
    // 禁止网络请求，缓存存在，返回响应到上一级
    if (networkRequest == null) {
      // return Cache
      ...
    }
    
    Response networkResponse = null;
    try {
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // 处理网络缓存失败时，释放缓存流
    }

    // 本地存在缓存则检查响应状态码是否为304
    if (cacheResponse != null) {
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        // update cache
        ...
        return response;
      } else {
        // close cacheResponse 
      }
    }
   
    // 构建新响应
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    if (HttpHeaders.hasBody(response)) {
      // 响应缓存
      CacheRequest cacheRequest = maybeCache(response, networkResponse.request(), cache);
      response = cacheWritingResponse(cacheRequest, response);
    }

    return response;
}
```

#### ConnectInterceptor
- 创建网络读写流必要的HttpCodec，用于请求编码和网络响应解码处理。
- 复用或建立Socket连接RealConnection，用于网络数据传输。
- 网络数据流具体处理细节交给CallServerInterceptor节点。

```java
@Override
public Response intercept(Chain chain) throws IOException {
    ... 
    StreamAllocation streamAllocation = realChain.streamAllocation();
    ...
    // 复用或创建新的RealConnection，并创建新的HttpCodec处理网络读写流。
    HttpCodec httpCodec = streamAllocation.newStream(client, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();
    // 交给CallServerInterceptor处理网络流。
    return realChain.proceed(request, streamAllocation, httpCodec, connection);
}
```

#### CallServerInterceptor
创建服务端的网络调用，向服务端发送请求并获取响应。

```java
@Override 
public Response intercept(Chain chain) throws IOException {
    
    // 获取需要写请求和读响应的HttpCodec
    ... 
    // 向服务端发送头部请求
    ...
    Response.Builder responseBuilder = null;
    // 如果含有支持的方法请求体，则需要向服务端发送请求体
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      // 发送请求体
      ...
    }
    // 结束请求
    httpCodec.finishRequest();
    
    // 如果头部响应未读取，则读取头部响应
    if (responseBuilder == null) {
      responseBuilder = httpCodec.readResponseHeaders(false);
    }

    // 构建响应体
    ... 
    return response;
}
```
#### 自定义Interceptor
OkHttp3除了默认5种Interceptor实现，还可以添加Application层与Network层的interceptor。

```java
/**
 * 返回一个可修改的拦截器列表，它们观察每个调用的全部范围：从链接建立之前（如果有的话），直到选择了响应源（远程服务器，缓存或两者）为止。
 */
public List<Interceptor> interceptors() {
  return interceptors;
}

public Builder addInterceptor(Interceptor interceptor) {
  interceptors.add(interceptor);
  return this;
}

/**
 * 返回观察单个网络请求和响应的可修改的拦截器列表。这些拦截器必须仅调用一次{@link Interceptor.Chain#proceed}：对于网络拦截器，发生短路或重复网络请求是一个错误。
 */
public List<Interceptor> networkInterceptors() {
  return networkInterceptors;
}

public Builder addNetworkInterceptor(Interceptor interceptor) {
  networkInterceptors.add(interceptor);
  return this;
}
```

1. Application Interceptors
    
    Application Interceptors对每个请求只调用一次，处理BridgeInterceptor返回的响应。
    可以不调用Chain.proceed()或多次调用Chain.proceed()。
    
2. 网络层Interceptors

    Network Interceptors在ConnectInterceptor与CallServerInterceptor之间调用。
    涉及到网络相关操作都会经过Network Interceptors，因此可以在缓存响应数据之前对响应数据进行预处理。
    与Application Interceptors不同的是不支持短路处理，必须且只能调用一次Chain.proceed()方法，保证链式调用唯一。
