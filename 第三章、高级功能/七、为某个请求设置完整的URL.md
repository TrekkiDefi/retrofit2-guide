假如说你的某一个请求不是以base_url开头该怎么办呢？别着急，办法很简单，看下面这个例子你就懂了
```java
public interface BlueService {  
    @GET
    public Call<ResponseBody> profilePicture(@Url String url);
}
```
```java
Retrofit retrofit = Retrofit.Builder()  
    .baseUrl("https://your.api.url/");
    .build();

BlueService service = retrofit.create(BlueService.class);  
service.profilePicture("https://s3.amazon.com/profile-picture/path");
```
直接用@Url注解的方式传递完整的url地址即可。
