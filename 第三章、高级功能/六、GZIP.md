### 压缩请求体
```java
/**
* 该拦截器压缩HTTP请求体。许多Web服务器无法处理这个！ 
*/
final class GzipRequestInterceptor implements Interceptor {
    @Override 
    public Response intercept(Chain chain) throws IOException {
        Request originalRequest = chain.request();
        
        // 如果请求体为空，或者已经指定请求体的编码，则不处理
        if (originalRequest.body() == null || originalRequest.header("Content-Encoding") != null) {
            return chain.proceed(originalRequest);
        }
        
        Request compressedRequest = originalRequest.newBuilder()
            .header("Content-Encoding", "gzip")
            .method(originalRequest.method(), gzip(originalRequest.body()))
            .build();
        return chain.proceed(compressedRequest);    
    }
    
    private RequestBody gzip(final RequestBody body) {
        return new RequestBody() {
            @Override
            public MediaType contentType() {
                return body.contentType();
            }
            @Override 
            public long contentLength() {
                return -1; // 我们预先不知道压缩长度!
            }
            @Override 
            public void writeTo(BufferedSink sink) throws IOException {
                BufferedSink gzipSink = Okio.buffer(new GzipSink(sink));
                body.writeTo(gzipSink);
                gzipSink.close();
            }
        };
    }
}
```
### 解压响应体
//TODO