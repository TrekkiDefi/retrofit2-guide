这里针对Okhttp3.6.X说明。

**Interceptors**是OkHttp3整个框架的核心，包含了请求监控、请求重写、调用重试等机制。它主要使用责任链模式，解决请求与请求处理之间的耦合。

![](/assets/okhttp3_process.png)