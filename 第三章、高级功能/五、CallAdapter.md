### RxJava与CallAdapter
说到Retrofit就不得说到另一个火到不行的库`RxJava`，网上已经不少文章讲如何与Retrofit结合，但这里还是会有一个RxJava的例子，
不过这里主要目的是介绍使用`CallAdapter`所带来的效果。

第3节介绍的`Converter`是对于`Call<T>`中`T`的转换，而`CallAdapter`则可以对`Call`转换，这样的话`Call<T>`中的`Call`也是可以被替换的，
而返回值的类型就决定你后续的处理程序逻辑，同样`Retrofit`提供了多个`CallAdapter`，这里以`RxJava`的为例，用`Observable`代替`Call`：

引入RxJava支持:

Maven:
```
<dependency>
    <groupId>com.squareup.retrofit2</groupId>
    <artifactId>adapter-rxjava</artifactId>
    <version>2.2.0</version>
</dependency>
```
通过RxJavaCallAdapterFactory为Retrofit添加RxJava支持：
```
Retrofit retrofit = new Retrofit.Builder()
      .baseUrl("http://localhost:4567/")
      .addConverterFactory(GsonConverterFactory.create())
      .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
      .build();
```

>源码参考：GsonConverterFactory.create()

```
public static GsonConverterFactory create() {
        return create(new Gson());
    }

    public static GsonConverterFactory create(Gson gson) {
        return new GsonConverterFactory(gson);
    }
```
接口设计：
```
public interface BlogService {
  @POST("/blog")
  Observable<Result<List<Blog>>> getBlogs();
}
```
使用：
```
BlogService service = retrofit.create(BlogService.class);
service.getBlogs(1)
  .observeOn(Schedulers.io())
  .subscribe(new Subscriber<Result<List<Blog>>>() {
      @Override
      public void onCompleted() {
        System.out.println("onCompleted");
      }

      @Override
      public void onError(Throwable e) {
        System.err.println("onError");
      }

      @Override
      public void onNext(Result<List<Blog>> blogsResult) {
        System.out.println(blogsResult);
      }
  });
```
结果：
```
Result{code=200, msg='OK', data=[Blog{id=1, date='2016-04-15 03:17:50', author='怪盗kidou', title='Retrofit2 测试1', content='这里是 Retrofit2 Demo 测试服务器1'},.....], count=20, page=1}
```

>像上面的这种情况最后我们无法获取到返回的Header和响应码的，如果我们需要这两者，提供两种方案：
>1. 用`Observable<Response<T>>`代替`Observable<T>` ,这里的`Response`指`retrofit2.Response`
>2. 用`Observable<Result<T>>` 代替`Observable<T>`，这里的`Result`是指`retrofit2.adapter.rxjava.Result`,这个Result中包含了Response的实例

### 自定义CallAdapter
本节将介绍如何自定一个`CallAdapter`，并验证是否所有的String都会使用我们第5节中自定义的Converter。

先看一下CallAdapter接口定义及各方法的作用：
```
public interface CallAdapter<T> {

  // 真正数据的类型 如Call<R> 中的 R
  // 这个 R 会作为Converter.Factory.responseBodyConverter 的第一个参数
  // 可以参照上面的自定义Converter
  Type responseType();

  <R> T adapt(Call<R> call);

  // 用于向Retrofit提供CallAdapter的工厂类
  abstract class Factory {
    // 在这个方法中判断是否是我们支持的类型，returnType 即Call<R>和Observable<R>
    // RxJavaCallAdapterFactory 就是判断returnType是不是Observable<?> 类型
    // 不支持时返回null
    public abstract CallAdapter<?> get(Type returnType, Annotation[] annotations,
    Retrofit retrofit);

    // 用于获取泛型的参数 如 Call<R> 中 R，CustomCall<R>中的R
    protected static Type getParameterUpperBound(int index, ParameterizedType type) {
      return Utils.getParameterUpperBound(index, type);
    }

    // 用于获取泛型的原始类型 如 Call<R> 中的 Call，CustomCall<R>中的CustomCall
    // 上面的get方法需要使用该方法。
    protected static Class<?> getRawType(Type type) {
      return Utils.getRawType(type);
    }
  }
}
```
了解了`CallAdapter`的结构和其作用之后，我们就可以开始自定义我们的`CallAdapter`了，本节以`CustomCall<String>`为例。

在此我们需要定义一个`CustomCall`，不过这里的`CustomCall`作为演示只是对`Call`的一个包装，并没有实际的用途。
```
public static class CustomCall<R> {

  public final Call<R> call;

  public CustomCall(Call<R> call) {
    this.call = call;
  }

  public R get() throws IOException {
    return call.execute().body();
  }
}
```
有了`CustomCall`，我们还需要一个`CustomCallAdapter`来实现 `Call<T>` 到 `CustomCall<T>`的转换，这里需要注意的是最后的泛型，是我们要返回的类型。
```
public static class CustomCallAdapter implements CallAdapter<CustomCall<?>> {

  private final Type responseType;

  // 下面的 responseType 方法需要数据的类型
  CustomCallAdapter(Type responseType) {
    this.responseType = responseType;
  }

  @Override
  public Type responseType() {
    return responseType;
  }

  @Override
  public <R> CustomCall<R> adapt(Call<R> call) {
    // 由 CustomCall 决定如何使用
    return new CustomCall<>(call);
  }
}
```
提供一个`CustomCallAdapterFactory`用于向Retrofit提供`CustomCallAdapter`：
```
public static class CustomCallAdapterFactory extends CallAdapter.Factory {
  public static final CustomCallAdapterFactory INSTANCE = new CustomCallAdapterFactory();

  @Override
  public CallAdapter<?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    // 获取原始类型
    Class<?> rawType = getRawType(returnType);
    // 返回值必须是CustomCall并且带有泛型
    if (rawType == CustomCall.class && returnType instanceof ParameterizedType) {
      Type responseType = getParameterUpperBound(0, (ParameterizedType) returnType);
      return new CustomCallAdapter(responseType);
    }
    return null;
  }
}
```
使用`addCallAdapterFactory`向Retrofit注册`CustomCallAdapterFactory`
```
Retrofit retrofit = new Retrofit.Builder()
      .baseUrl("http://localhost:4567/")
      .addConverterFactory(Example09.StringConverterFactory.create())
      .addConverterFactory(GsonConverterFactory.create())
      .addCallAdapterFactory(CustomCallAdapterFactory.INSTANCE)
      .build();
```
注：`addCallAdapterFactory`与`addConverterFactory`同理，也有先后顺序。