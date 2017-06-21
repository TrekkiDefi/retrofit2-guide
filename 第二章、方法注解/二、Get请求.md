### @Query
```java
public class TokenInterceptor implements Interceptor {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        HttpUrl httpUrl = request.url().newBuilder()
                .addQueryParameter("token", "tokenValue")
                .build();
        request = request.newBuilder().url(httpUrl).build();
        return chain.proceed(request);
    }
}
```
addQueryParameter就是添加请求参数的具体代码，这种方式比较适用于所有的请求都需要添加的参数，一般现在的网络请求都会添加token作为用户标识，那么这种方式就比较适合。

创建完成自定义的Interceptor后，还需要在Retrofit创建client处完成添加
```java
addInterceptor(new TokenInterceptor())
```
### @QueryMap
如果Query参数比较多，那么可以通过@QueryMap方式将所有的参数集成在一个Map统一传递，还以上文中的get请求方法为例
```java
public interface BlueService {
    @GET("book/search")
    Call<BookSearchResponse> getSearchBooks(@QueryMap Map<String, String> options);
}
```
调用的时候将所有的参数集合在统一的map中即可
```java
Map<String, String> options = new HashMap<>();
map.put("q", "小王子");
map.put("tag", null);
map.put("start", "0");
map.put("count", "3");
Call<BookSearchResponse> call = mBlueService.getSearchBooks(options);
```
### Query集合
假如你需要添加相同Key值，但是value却有多个的情况，一种方式是添加多个@Query参数，还有一种简便的方式是将所有的value放置在列表中，然后在同一个@Query下完成添加，实例代码如下：
```java
public interface BlueService {
    @GET("book/search")
    Call<BookSearchResponse> getSearchBooks(@Query("q") List<String> name);
}
```
最后得到的url地址为
```text
https://api.douban.com/v2/book/search?q=leadership&q=beyond%20feelings
```
如果请求参数为非必填，也就是说即使不传该参数，服务端也可以正常解析，那么如何实现呢？ 其实也很简单，请求方法定义处还是需要完整的Query注解，某次请求如果不需要传该参数的话，只需填充null即可。

针对文章开头提到的get的请求，加入按以下方式调用
```java
Call<BookSearchResponse> call = mBlueService.getSearchBooks("小王子", null, 0, 3);
```
那么得到的url地址为
```text
https://api.douban.com/v2/book/search?q=%E5%B0%8F%E7%8E%8B%E5%AD%90&start=0&count=3
```
### @Path
如果请求的相对地址也是需要调用方传递，那么可以使用@Path注解，示例代码如下：
```java
@GET("book/{id}")
Call<BookResponse> getBook(@Path("id") String id);
```
业务方想要在地址后面拼接书籍id，那么通过Path注解可以在具体的调用场景中动态传递，具体的调用方式如下：
```text
Call<BookResponse> call = mBlueService.getBook("1003078");
```
此时的url地址为
```text
https://api.douban.com/v2/book/1003078
```
>@Path可以用于任何请求方式，包括Post，Put，Delete等等
