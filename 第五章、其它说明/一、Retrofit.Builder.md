前面用到了 `Retrofit.Builder` 中的`baseUrl`、`addCallAdapterFactory`、`addConverterFactory`、`build`方法， 
还有`callbackExecutor`、`callFactory`、`client`、`validateEagerly`这四个方法没有用到，这里简单的介绍一下。

|方法     |用途     |
|-------------|-------------|
|callbackExecutor(Executor)|指定`Call.enqueue`时使用的`Executor`，所以该设置只对返回值为`Call`的方法有效|
|callFactory(Factory)|设置一个自定义的`okhttp3.Call.Factory`，那什么是`Factory`呢?`OkHttpClient`就实现了`okhttp3.Call.Factory`接口，下面的`client(OkHttpClient)`最终也是调用了该方法，也就是说两者不能共用|
|client(OkHttpClient)|设置自定义的`OkHttpClient`,以前的Retrofit版本中不同的`Retrofit`对象共用同`OkHttpClient`,在2.0各对象各自持有不同的`OkHttpClient`实例，所以当你需要共用`OkHttpClient`或需要自定义时则可以使用该方法.
|validateEagerly(boolean)|是否在调用"create(Class)"时检测接口定义是否正确，而不是在调用方法才检测，适合在开发、测试时使用|

>源码参考：

```java
public class OkHttpClient implements Cloneable, Factory, okhttp3.WebSocket.Factory {
    ...
}
```
```java
public Retrofit.Builder client(OkHttpClient client) {
    return this.callFactory((Factory)Utils.checkNotNull(client, "client == null"));
}
```