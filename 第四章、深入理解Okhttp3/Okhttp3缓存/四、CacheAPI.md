这里看下类CacheControl、Cache的相关注解说明。

>源码参考：okhttp3.CacheControl.java

```java
package okhttp3;
/**
* 具有来自服务器或客户端的缓存指令的Cache-Control Header。
* 这些指令设定了可以存储哪些响应哪些请求可以由那些存储的响应来满足的策略。
*  
* <p>请参见<a href="http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.9">RFC 2616, 14.9</a>。
*/
public final class CacheControl {
    
    /**
    * 需要网络验证响应的缓存控制请求指令。请注意，这些请求可以通过 {@code conditional GET requests} 由缓存来辅助。
    * 
    * 关于 {@code conditional GET requests} 的描述参见【源码参考：okhttp3.Cache.java】部分。
    */
    public static final CacheControl FORCE_NETWORK = new Builder().noCache().build();
    
    /**
    * 缓存控制请求指令仅使用缓存，即使缓存响应过时。
    * 如果缓存中响应不可用或需要服务器验证，则调用将失败，返回504状态码。
    */
    public static final CacheControl FORCE_CACHE = new Builder()
          .onlyIfCached()
          .maxStale(Integer.MAX_VALUE, TimeUnit.SECONDS)
          .build();
    
    ...
    
    /**
    * 在响应中，此字段的名称"无缓存"是误导性的。
    * 它不阻止我们缓存响应数据; 这只意味着我们必须在返回响应之前验证源服务器的响应。
    * 我们可以使用 {@code a conditional GET} 来执行此操作。
    * <p>在请求中，这意味着不要使用缓存来满足请求。
    * @return 
    */
    public boolean noCache() {
        return noCache;
    }
    
    /**
    * 该字段的名称"only-if-cached"是误导性的。它实际上意味着"不要使用网络"。<br>
    * 它由客户端设置，只有当缓存完全满足时才想要发出请求。<br>
    * 如果设置了此头，则不允许需要验证的缓存响应(ie. conditional gets)。<br>
    * @return 
    */
    public boolean onlyIfCached() {
        return onlyIfCached;
    }
    
    ...
    
    /**
    * 返回{@code headers}的缓存指令。如果它们存在，这支持Cache-Control和Pragma headers。
    * 
    * 这里我们知道了，上面实现缓存的时候为什么我们需要移除Pragma响应头。
    */
    public static CacheControl parse(Headers headers) {
      
    }
    
    ...
    
    public static final class Builder {
        
        /**
        * 设置一个缓存响应的最大年龄。如果缓存响应的年龄超过{@code maxAge}，则不会使用缓存响应，将发出一个网络请求。
        * @param maxAge 一个非负整数。这是使用{@link TimeUnit#SECONDS}精度存储和传输的; 更精确的精度将会丢失。
        */
        public Builder maxAge(int maxAge, TimeUnit timeUnit) {
              if (maxAge < 0) throw new IllegalArgumentException("maxAge < 0: " + maxAge);
              long maxAgeSecondsLong = timeUnit.toSeconds(maxAge);
              this.maxAgeSeconds = maxAgeSecondsLong > Integer.MAX_VALUE
                  ? Integer.MAX_VALUE
                  : (int) maxAgeSecondsLong;
              return this;
        }
        
        /**
        * 接受已经超过了他们的 {@code freshness lifetime} 达{@code maxStale}的缓存响应。如果未指定，则不会使用 {@code stale cache responses} 。
        * @param maxStale 一个非负整数。这是使用{@link TimeUnit#SECONDS}精度存储和传输的; 更精确的精度将会丢失。
        */
        public Builder maxStale(int maxStale, TimeUnit timeUnit) {
              if (maxStale < 0) throw new IllegalArgumentException("maxStale < 0: " + maxStale);
              long maxStaleSecondsLong = timeUnit.toSeconds(maxStale);
              this.maxStaleSeconds = maxStaleSecondsLong > Integer.MAX_VALUE
                  ? Integer.MAX_VALUE
                  : (int) maxStaleSecondsLong;
              return this;
        }
        
        /**
        * 设置响应将继续保持最新的秒数。
        * 如果在{@code minFresh}过后响应将过时，缓存的响应将不会被使用，并且将进行网络请求。
        * @param minFresh 一个非负整数。这是使用{@link TimeUnit#SECONDS}精度存储和传输的; 更精确的精度将会丢失。
        */
        public Builder minFresh(int minFresh, TimeUnit timeUnit) {
              if (minFresh < 0) throw new IllegalArgumentException("minFresh < 0: " + minFresh);
              long minFreshSecondsLong = timeUnit.toSeconds(minFresh);
              this.minFreshSeconds = minFreshSecondsLong > Integer.MAX_VALUE
                  ? Integer.MAX_VALUE
                  : (int) minFreshSecondsLong;
              return this;
        }
    }
}
```

>源码参考：okhttp3.Cache.java

```java
package okhttp3;

/**
* 缓存HTTP和HTTPS响应到文件系统中，以便它们可以重用，从而节省时间和带宽。
* 
* <h3>缓存优化</h3>
* 
* <p>为了衡量缓存的有效性，这个类跟踪三个统计信息：
* <ul>
*       <li> <strong>{@linkplain #requestCount() Request Count:}</strong>创建该缓存以来的HTTP请求数。
*       <li> <strong>{@linkplain #networkCount() Network Count:}</strong>需要使用网络的请求数。
*       <li> <strong>{@linkplain #hitCount() Hit Count:}</strong>缓存提供响应的请求数。
* </ul>
* 有时，请求将导致 {@code a conditional cache hit}。
* 如果缓存包含 {@code a stale copy of the response} ，客户端将发出 {@code a conditional GET} 。
* 服务器将发送更新的响应（如果已更改）或简短的的'not modified'响应(如果客户端的副本仍然有效)。
* 这样的响应会增加网络计数和命中次数。
*  
* <p>提高缓存命中率的最佳方法是配置Web服务器以返回可缓存的响应。
* 尽管此客户端遵从所有<a href="http://tools.ietf.org/html/rfc7234">HTTP/1.1 (RFC 7234)</a>缓存头，但它不缓存部分响应。
*  
* <h3>强制网络响应</h3>
* <p>在某些情况下，例如用户点击"刷新"按钮后，可能需要跳过缓存，并直接从服务器获取数据。
* 要强制完全刷新，请添加{@code no-cache} 指令：
* <pre>{@code
*     Request request = new Request.Builder()
*         .cacheControl(new CacheControl.Builder().noCache().build())
*         .url("http://publicobject.com/helloworld.txt")
*         .build();
* }</pre>
* 
* 如果仅需要强制一个缓存的响应由服务器验证，则使用更有效的{@code max-age=0}指令：
* <pre>{@code
*     Request request = new Request.Builder()
*         .cacheControl(new CacheControl.Builder()
*             .maxAge(0, TimeUnit.SECONDS)
*             .build())
*         .url("http://publicobject.com/helloworld.txt")
*         .build();
* }</pre>
* 
* <h3>强制缓存响应</h3>
* <p>有时你会想显示可以立即显示的资源。
* 这是可以使用的，这样你的应用程序可以在等待最新的数据下载的时候显示一些东西， 
* 重定向请求到本地缓存资源，请添加{@code only-if-cached}指令：
* <pre>   {@code
*     Request request = new Request.Builder()
*         .cacheControl(new CacheControl.Builder()
*             .onlyIfCached()
*             .build())
*         .url("http://publicobject.com/helloworld.txt")
*         .build();
*     Response forceCacheResponse = client.newCall(request).execute();
*     if (forceCacheResponse.code() != 504) {
*       // 资源缓存命中，显示它
*     } else {
*       // 资源缓存未命中，即缓存不存在该资源
*     }
* }</pre>
* 
* 这一技术在 {@code a stale response} 比没有响应更好的情况下运行得更好。
* 要允许 {@code stale cached responses} ，请使用{@code max-stale}指令，指定其最长过期时间，单位为秒：
* 
* <pre>   {@code
*   Request request = new Request.Builder()
*       .cacheControl(new CacheControl.Builder()
*           .maxStale(365, TimeUnit.DAYS)
*           .build())
*       .url("http://publicobject.com/helloworld.txt")
*       .build();
* }</pre>
*  
* <p> {@link CacheControl}类可以配置请求缓存指令并解析响应缓存指令。 
* 它甚至提供了方便的常量{@link CacheControl#FORCE_NETWORK}和{@link CacheControl#FORCE_CACHE}，用于解决上述用例。
*/
public final class Cache implements Closeable, Flushable {
    ...
    
    CacheRequest put(Response response) {
        String requestMethod = response.request().method();
    
        if (HttpMethod.invalidatesCache(response.request().method())) {
          try {
            remove(response.request());
          } catch (IOException ignored) {
            // The cache cannot be written.
          }
          return null;
        }
        if (!requestMethod.equals("GET")) {
          // 不缓存非GET响应。 我们在技术上允许缓存HEAD请求和一些POST请求，但是这样做的复杂性很高，而且效益也很低。
          // 这里我们可以看出，只缓存Get响应的数据
          return null;
        }
    
        if (HttpHeaders.hasVaryAll(response)) {
          return null;
        }
    
        Entry entry = new Entry(response);
        DiskLruCache.Editor editor = null;
        try {
          editor = cache.edit(key(response.request().url()));
          if (editor == null) {
            return null;
          }
          entry.writeTo(editor);
          return new CacheRequestImpl(editor);
        } catch (IOException e) {
          abortQuietly(editor);
          return null;
        }
      }
      
      ...
}
```
