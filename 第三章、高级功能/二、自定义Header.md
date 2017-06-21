Retrofit提供了两个方式定义Http请求头参数：静态方法和动态方法， 静态方法不能随不同的请求进行变化，头部信息在初始化的时候就固定了。 而动态方法则必须为每个请求都要单独设置。

### 静态方法
```java
public interface BlueService {
@Headers("Cache-Control: max-age=640000")
  @GET("book/search")
  Call<BookSearchResponse> getSearchBooks(@Query("q") String name, 
        @Query("tag") String tag, @Query("start") int start, 
        @Query("count") int count);
}
```
当然你想添加多个header参数也是可以的，写法也很简单
```java
public interface BlueService {
@Headers({
      "Accept: application/vnd.yourapi.v1.full+json",
      "User-Agent: Your-App-Name"
  })
  @GET("book/search")
  Call<BookSearchResponse> getSearchBooks(@Query("q") String name, 
        @Query("tag") String tag, @Query("start") int start, 
        @Query("count") int count);
}
```
#### 通过Interceptor定义静态请求头
```java
public class RequestInterceptor implements Interceptor {
  @Override
  public Response intercept(Chain chain) throws IOException {
      Request original = chain.request();
      Request request = original.newBuilder()
          .header("User-Agent", "Your-App-Name")
          .header("Accept", "application/vnd.yourapi.v1.full+json")
          .method(original.method(), original.body())
          .build();
      return chain.proceed(request);
  }
}
```

>源码参考：okhttp3.Request.java

- request.newBuilder()：
```java
public Request.Builder newBuilder() {
    return new Request.Builder(this);
}
```
```java
Builder(Request request) {
    this.url = request.url;
    this.method = request.method;
    this.body = request.body;
    this.tag = request.tag;
    this.headers = request.headers.newBuilder();
}
```
- request.headers.newBuilder：
```java
public Headers.Builder newBuilder() {
    Headers.Builder result = new Headers.Builder();
    Collections.addAll(result.namesAndValues, this.namesAndValues);
    return result;
}
```
由此可见，newBuilder()方法根据原有的Request对象创建新的Request对象

Request提供了两个方法添加header参数，一个是header(key, value)，另一个是.addHeader(key, value)， 两者的区别是，header()如果有重名的将会覆盖，而addHeader()允许相同key值的header存在

然后在OkHttp创建Client实例时，添加RequestInterceptor即可
```java
new OkHttpClient.Builder()
  .addInterceptor(new RequestInterceptor())
  .connectTimeout(DEFAULT_TIMEOUT, TimeUnit.SECONDS)
  .build();
```

### 动态方法
```java
public interface BlueService {
  @GET("book/search")
  Call<BookSearchResponse> getSearchBooks(
  @Header("Content-Range") String contentRange, 
  @Query("q") String name, @Query("tag") String tag, 
  @Query("start") int start, @Query("count") int count);
}
```