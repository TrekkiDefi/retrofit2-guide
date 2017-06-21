### @Field
```java
@FormUrlEncoded
@POST("book/reviews")
Call<String> addReviews(@Field("book") String bookId, @Field("title") String title,
@Field("content") String content, @Field("rating") String rating);
```
这里有几点需要说明的:
1. `@FormUrlEncoded`将会自动将请求的类型调整为`application/x-www-form-urlencoded`，
    假如content传递的参数为Good Luck，那么最后得到的请求体就是
    ```text
    content=Good+Luck
    ```
    >FormUrlEncoded不能用于Get请求
    
2. @Field注解将每一个请求参数都存放至请求体中，还可以添加encoded参数，该参数为boolean型，具体的用法为
    ```text
    @Field(value = "book", encoded = true) String book
    ```
    encoded参数为true的话，key-value-pair将会被编码，即将中文和特殊字符进行编码转换
    
### @FieldMap
上述Post请求有4个请求参数，假如说有更多的请求参数，那么通过一个一个的参数传递就显得很麻烦而且容易出错，这个时候就可以用FieldMap
```java
@FormUrlEncoded
@POST("book/reviews")
Call<String> addReviews(@FieldMap Map<String, String> fields);
```
### @Body
如果Post请求参数有多个，那么统一封装到类中应该会更好，这样维护起来会非常方便
```java
@POST("book/reviews")
Call<String> addReviews(@Body Reviews reviews);

```
```java
public class Reviews {
    public String book;
    public String title;
    public String content;
    public String rating;
}
```