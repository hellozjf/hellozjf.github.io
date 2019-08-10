# 第六章 HTTP缓存

## 6.1. 一般概念

HttpClient Cache提供了一个与HttpClient一起使用的HTTP / 1.1兼容缓存层 - 相当于浏览器缓存的Java。 实现遵循Chain of Responsibility设计模式，其中缓存HttpClient实现可以为默认的非缓存HttpClient实现提供替代; 完全可以从缓存中满足的请求不会导致实际的原始请求。 使用条件GET和If-Modified-Since和/或If-None-Match请求标头，尽可能使用原点自动验证过时的缓存条目。

HTTP / 1.1缓存通常设计为语义透明; 也就是说，缓存不应该改变客户端和服务器之间的请求 - 响应交换的含义。 因此，将缓存HttpClient放入现有的兼容客户端 - 服务器关系应该是安全的。 虽然从HTTP协议的角度来看，缓存模块是客户端的一部分，但实现的目的是与透明缓存代理上的要求兼容。

最后，缓存HttpClient包括支持RFC 5861指定的Cache-Control扩展（stale-if-error和stale-while-revalidate）。

当缓存HttpClient执行请求时，它会经历以下流程：

* 检查基本符合HTTP 1.1协议的请求，并尝试更正请求。
* 刷新将被此请求无效的任何缓存条目。
* 确定当前请求是否可以从缓存中获取。 如果没有，直接将请求传递给原始服务器并在适当时缓存后返回响应。
* 如果它是一个可缓存服务的请求，它将尝试从缓存中读取它。 如果它不在缓存中，请调用原始服务器并缓存响应（如果适用）。
* 如果缓存的响应适合作为响应，请构造包含ByteArrayEntity的BasicHttpResponse并将其返回。 否则，尝试针对源服务器重新验证缓存条目。
* 对于无法重新验证的缓存响应，请调用源服务器并缓存响应（如果适用）。

当缓存HttpClient收到响应时，它会经历以下流程：

* 检查协议合规性的响应
* 确定响应是否可缓存
* 如果它是可缓存的，请尝试读取配置中允许的最大大小并将其存储在缓存中。
* 如果响应对于缓存而言太大，则重新构建部分消耗的响应并直接返回它而不缓存它。

值得注意的是，缓存HttpClient本身并不是HttpClient的不同实现，但它通过将自身作为附加处理组件插入到请求执行管道中来工作。

## 6.2. RFC-2616合规性

我们相信HttpClient Cache无条件地符合RFC-2616。 也就是说，无论规范指示MUST，MUST NOT，SHOULD或者不应该是HTTP缓存，缓存层都会尝试以满足这些要求的方式运行。 这意味着当您将其放入时，缓存模块不会产生不正确的行为。

## 6.3. 示例用法

这是一个如何设置基本缓存HttpClient的简单示例。 根据配置，它将存储最多1000个缓存对象，每个缓存对象的最大主体大小可以为8192个字节。 此处选择的数字仅作为示例，并非旨在作为规定或被视为建议。

```
CacheConfig cacheConfig = CacheConfig.custom()
        .setMaxCacheEntries(1000)
        .setMaxObjectSize(8192)
        .build();
RequestConfig requestConfig = RequestConfig.custom()
        .setConnectTimeout(30000)
        .setSocketTimeout(30000)
        .build();
CloseableHttpClient cachingClient = CachingHttpClients.custom()
        .setCacheConfig(cacheConfig)
        .setDefaultRequestConfig(requestConfig)
        .build();
 
HttpCacheContext context = HttpCacheContext.create();
HttpGet httpget = new HttpGet("http://www.mydomain.com/content/");
CloseableHttpResponse response = cachingClient.execute(httpget, context);
try {
    CacheResponseStatus responseStatus = context.getCacheResponseStatus();
    switch (responseStatus) {
        case CACHE_HIT:
            System.out.println("A response was generated from the cache with " +
                    "no requests sent upstream");
            break;
        case CACHE_MODULE_RESPONSE:
            System.out.println("The response was generated directly by the " +
                    "caching module");
            break;
        case CACHE_MISS:
            System.out.println("The response came from an upstream server");
            break;
        case VALIDATED:
            System.out.println("The response was generated from the cache " +
                    "after validating the entry with the origin server");
            break;
    }
} finally {
    response.close();
}
```

## 6.4. 配置

缓存HttpClient继承了默认非缓存实现的所有配置选项和参数（这包括设置超时和连接池大小等选项）。 对于特定于缓存的配置，您可以提供CacheConfig实例以自定义以下方面的行为：

缓存大小。 如果后端存储支持这些限制，则可以指定最大缓存条目数以及最大可缓存响应主体大小。

公共/私人缓存。 默认情况下，缓存模块将自身视为共享（公共）缓存，并且不会缓存对具有授权标头或标记为“Cache-Control：private”的响应的请求的响应。 但是，如果缓存仅由一个逻辑“用户”使用（表现类似于浏览器缓存），那么您将需要关闭共享缓存设置。

启发式缓存.Per RFC2616，缓存可以缓存某些缓存条目，即使源没有设置显式缓存控制头。 默认情况下，此行为处于关闭状态，但如果您正在使用未设置正确标头但仍希望缓存响应的原点，则可能需要启用此功能。 您将需要启用启发式缓存，然后指定默认的新鲜度生命周期和/或自上次修改资源以来的一小部分时间。 有关启发式缓存的更多详细信息，请参阅HTTP / 1.1 RFC的第13.2.2节和第13.2.4节。

背景验证。 缓存模块支持RFC5861的stale-while-revalidate指令，该指令允许某些缓存条目重新验证在后台发生。 您可能希望调整后台工作线程的最小和最大数量的设置，以及它们在回收之前可以空闲的最长时间。 当没有足够的工作人员来满足需求时，您还可以控制用于重新验证的队列的大小。

## 6.5. 存储后端

缓存HttpClient的默认实现将缓存条目和缓存的响应主体存储在应用程序的JVM的内存中。 虽然这提供了高性能，但由于大小限制或缓存条目是短暂的并且在应用程序重新启动后无法生存，因此它可能不适合您的应用程序。 当前版本包括支持使用EhCache和memcached实现存储缓存条目，这些实现允许将缓存条目溢出到磁盘或将它们存储在外部进程中。

如果这些选项都不适合您的应用程序，则可以通过实现HttpCacheStorage接口然后在构建时将其提供给缓存HttpClient来提供您自己的存储后端。 在这种情况下，将使用您的方案存储缓存条目，但您将重用所有围绕HTTP / 1.1合规性和缓存处理的逻辑。 一般来说，应该可以使用能够应用原子更新的任何支持键/值存储（类似于Java Map接口）的东西创建HttpCacheStorage实现。

最后，通过一些额外的努力，完全可以建立一个多层缓存层次结构; 例如，将内存缓存HttpClient包装在一个存储缓存条目的磁盘上或远程存储在memcached中，遵循类似于虚拟内存，L1 / L2处理器缓存等的模式。