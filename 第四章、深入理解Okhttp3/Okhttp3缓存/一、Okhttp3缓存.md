#### 设置OkHttpClient
```java
private static final OkHttpClient client;
private static final long cacheSize = 1024 * 1024 * 20;//缓存文件大小限制20M  
private static final String filePathRoot = ;
private static String cachedirectory = filePathRoot + "/caches";//设置缓存文件路径  
private static Cache cache = new Cache(new File(cachedirectory), cacheSize);
static {
    OkHttpClient.Builder builder = new OkHttpClient.Builder();
    builder.connectTimeout(8, TimeUnit.SECONDS);  //设置连接超时时间  
    builder.writeTimeout(8, TimeUnit.SECONDS);//设置写入超时时间  
    builder.readTimeout(8, TimeUnit.SECONDS);//设置读取数据超时时间  
    builder.retryOnConnectionFailure(false);//设置不进行连接失败重试  
    builder.cache(cache);//设置缓存  
    client = builder.build();  
}
```
>需要注意的是：Okhttp只会对**GET**请求进行缓存，**POST**请求是不会进行缓存

#### 设置缓存策略
一般来说的是，我们**GET**请求有时有不一样的需求，有时需要进行缓存，有时需要直接从网络获取，有时只获取缓存数据，
这些处理，okhttp都有帮我们做了，我们做的只需要设置就是了。

下面是整理的各种需求的设置与使用方法。

异步请求方法doRequest：
```java
private static void doRequest(final Call call0) {  
        try {  
            call0.enqueue(new Callback() {  
                @Override  
                public void onFailure(Call arg0, IOException arg1) {  
                    //请求失败  
                }  
                @Override  
                public void onResponse(Call arg0, Response response) throws IOException {  
                    //请求返回数据  
                }  
            });  
        } catch (Exception e) {  
        }  
    }  
```

**1.Get请求，缓存命中返回缓存响应，否则请求网络**
```java
/**
* Get请求，并缓存请求数据。
* 返回的是“缓存数据”，注意，如果超出了maxAge，“缓存数据会被清除”，会请求网络
* @param url
* @param cache_maxAge_inseconds 缓存最大生存时间，单位为秒
* @return 当前call
*/
public static Call doGetAndCache(String url, int cache_maxAge_inseconds) {
    Request request = new Request.Builder()
                .cacheControl(new CacheControl.Builder().maxAge(cache_maxAge_inseconds, TimeUnit.SECONDS).build())
                .url(url).build();
    
        Call call = client.newCall(request);
        doRequest(call);
        return call;
}
```
**2.Get请求，获取返回网络请求数据**
```java
/**
* Get请求 ,获取返回网络请求数据
* 
* @param url
* @param responseListener
*/
public static Call doGetOnlyNet(String url) {
    Request request = new Request.Builder()
                .cacheControl(new CacheControl.Builder().maxAge(0, TimeUnit.SECONDS).build())
                .url(url).build();
    
        Call call = client.newCall(request);
        doRequest(call);
        return call;
}
```

**3.Get请求, 没有超过过时时间StaleTime的话，返回缓存数据，否则重新去获取网络数据** 
```java
/**
* Get请求, 没有超过“过时时间”StaleTime的话，返回缓存数据，否则重新去获取网络数据
* StaleTime限制了默认数据fresh时间
* 
* @param url
* @param staletime 过时时间，秒为单位
*/
public static Call doGetInStaleTime(String url, int staletime) {
    Request request = new Request.Builder()
            .cacheControl(new CacheControl.Builder().maxStale(staletime, TimeUnit.SECONDS).build()).url(url).build();

    Call call = client.newCall(request);
    doRequest( call);
    return call;
}
```
**4.Get请求, 只使用缓存**
```java
/**
* Get请求, 只使用缓存，
* 注意，如果是超出了staletime或者超出了maxAge的话会返回504，否则就返回缓存数据
* 
* @param url
*/
public static Call doGetOnlyCache(String url) {
    Request request = new Request.Builder()
            .cacheControl(new CacheControl.Builder().onlyIfCached().build()).url(url).build();

    Call call = client.newCall(request);
    doRequest(call);
    return call;
```
此外还可以使用`CacheControl.FORCE_CACHE`

>源码参考：CacheControl.FORCE_CACHE

```java
static {
    FORCE_CACHE = (new CacheControl.Builder()).onlyIfCached().maxStale(2147483647, TimeUnit.SECONDS).build();//2147483647，它等于2^31-1，是32位操作系统中最大的符号型整型常量
}
```

下面测试一下，设置缓存maxAge为10秒，我们将请求返回的数据打印出来：
![](..\images\cache_test_log.png)

上面图片的第一行是点击获取的服务器的数据，获取后断开网络然后继续点击，可以看到后面三行还能获取到数据，说明这是缓存的数据，
最后三行，差不多就是十秒的时间，可以看到，获取数据失败了，这时已经去服务器获取数据了，缓存被清空，由于断开网络所以请求失败。
