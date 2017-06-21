### Gson与Converter
在默认情况下Retrofit只支持将HTTP的响应体转换换为`ResponseBody`,
这也是为什么我在前面的例子接口的返回值都是 `Call<ResponseBody>`，
但如果响应体只是支持转换为`ResponseBody`的话何必要引用泛型呢，
返回值直接用一个`Call`就行了嘛，既然支持泛型，那说明泛型参数可以是其它类型的，
而`Converter`就是Retrofit为我们提供用于将`ResponseBody`转换为我们想要的类型，
有了`Converter`之后我们就可以写把我们的第一个例子的接口写成这个样子了：
```
public interface BlogService {
  @GET("blog/{id}")
  Call<Result<Blog>> getBlog(@Path("id") int id);
}
```
当然只改变泛型的类型是不行的，我们在创建Retrofit时需要明确告知用于将`ResponseBody`转换我们泛型中的类型时需要使用的`Converter`

引入Gson支持:

Maven:
```
<dependency>
    <groupId>com.squareup.retrofit2</groupId>
    <artifactId>converter-gson</artifactId>
    <version>2.2.0</version>
</dependency>
```
通过GsonConverterFactory为Retrofit添加Gson支持：
```
Gson gson = new GsonBuilder()
      //配置你的Gson
      .setDateFormat("yyyy-MM-dd hh:mm:ss")
      .create();

Retrofit retrofit = new Retrofit.Builder()
      .baseUrl("http://localhost:4567/")
      //可以接收自定义的Gson，当然也可以不传
      .addConverterFactory(GsonConverterFactory.create(gson))
      .build();
```

这样Retrofit就会使用Gson将`ResponseBody`转换我们想要的类型。

这是时候我们终于可以演示如使创建一个Blog了！

```
@POST("blog")
Call<Result<Blog>> createBlog(@Body Blog blog);
```
被`@Body`注解的的Blog将会被Gson转换成RequestBody发送到服务器。
```
BlogService service = retrofit.create(BlogService.class);
Blog blog = new Blog();
blog.content = "新建的Blog";
blog.title = "测试";
blog.author = "ittalks";
Call<Result<Blog>> call = service.createBlog(blog);
call.enqueue(new Callback<Result<Blog>>() {
    @Override
    public void onResponse(Call<Result<Blog>> call, Response<Result<Blog>> response) {
        // 已经转换为想要的类型了
        Result<Blog> result = response.body();
        System.out.println(result);
    }

    @Override
    public void onFailure(Call<Result<Blog>> call, Throwable t) {
        t.printStackTrace();
    }
});
```
结果：
```
Result{code=200, msg='OK', data=Blog{id=20, date='2016-04-21 05:29:58', author='ittalks', title='测试', content='新建的Blog'}, count=0, page=0}
```
Result的方法： 
```
package com.github.ikidou.entity;

public class Result<T> {
    public int code;
    public String msg;
    public T data;
    public long count;
    public long page;

    @Override
    public String toString() {
        return "Result{" +
                "code=" + code +
                ", msg='" + msg + '\'' +
                ", data=" + data +
                ", count=" + count +
                ", page=" + page +
                '}';
    }
}
```

### 自定义Converter
本节的内容是教大家实现一简易的Converter，这里以返回格式为`Call<String>`为例。
在此之前先了解一下Converter接口及其作用：

```
public interface Converter<F, T> {
  // 实现从 F(rom) 到 T(o)的转换
  T convert(F value) throws IOException;

  // 用于向Retrofit提供相应Converter的工厂
  abstract class Factory {
    // 这里创建从ResponseBody到其它类型的Converter，如果不能处理返回null
    // 主要用于对响应体的处理
    public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
    Retrofit retrofit) {
      return null;
    }

    // 在这里创建 从自定类型到RequestBody的Converter,不能处理就返回null，
    // 主要用于对Part、PartMap、Body注解的处理
    public Converter<?, RequestBody> requestBodyConverter(Type type,
    Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
      return null;
    }

    // 这里用于对Field、FieldMap、Header、Path、Query、QueryMap注解的处理
    // Retrfofit对于上面的几个注解默认使用的是调用toString方法
    public Converter<?, String> stringConverter(Type type, Annotation[] annotations,
    Retrofit retrofit) {
      return null;
    }

  }
}
```
我们要想从`Call<ResponseBody>` 转换为 `Call<String>` 那么对应的F和T则分别对应`ResponseBody`和`String`，我们定义一个`StringConverter`并实现`Converter`接口。
```
public static class StringConverter implements Converter<ResponseBody, String> {

  public static final StringConverter INSTANCE = new StringConverter();

  @Override
  public String convert(ResponseBody value) throws IOException {
    return value.string();
  }
}
```
我们需要一个`Fractory`来向Retrofit注册`StringConverter`
```
public static class StringConverterFactory extends Converter.Factory {

  public static final StringConverterFactory INSTANCE = new StringConverterFactory();

  public static StringConverterFactory create() {
    return INSTANCE;
  }

  // 我们只管实现从ResponseBody 到 String 的转换，所以其它方法可不覆盖
  @Override
  public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations, Retrofit retrofit) {
    if (type == String.class) {
      return StringConverter.INSTANCE;
    }
    //其它类型我们不处理，返回null就行
    return null;
  }
}
```
使用`Retrofit.Builder.addConverterFactory`向Retrofit注册我们`StringConverterFactory`：
```
Retrofit retrofit = new Retrofit.Builder()
      .baseUrl("http://localhost:4567/")
      // 如是有Gson这类的Converter 一定要放在其前面
      .addConverterFactory(StringConverterFactory.create())
      .addConverterFactory(GsonConverterFactory.create())
      .build();
```
>注：`addConverterFactory`是有先后顺序的，如果有多个`ConverterFactory`都支持同一种类型，那么就是只有第一个才会被使用，
而`GsonConverterFactory`是不判断是否支持的，所以如果这里交换了顺序，则会有一个异常抛出，原因是类型不匹配。

只要返回值类型的泛型参数是String类型，就会由我们的`StringConverter`处理,不管是`Call<String>`还是`Observable<String>`
