# Chapter 6. HTTP Caching

## 6.1. General Concepts

HttpClient Cache provides an HTTP/1.1-compliant caching layer to be used with HttpClient--the Java equivalent of a browser cache. The implementation follows the Chain of Responsibility design pattern, where the caching HttpClient implementation can serve a drop-in replacement for the default non-caching HttpClient implementation; requests that can be satisfied entirely from the cache will not result in actual origin requests. Stale cache entries are automatically validated with the origin where possible, using conditional GETs and the If-Modified-Since and/or If-None-Match request headers.

HttpClient Cache 提供了一个 HTTP/1.1 兼容的缓存层，用于 HttpClient（相当于浏览器缓存的 Java 版本）。其实现遵循责任链设计模式，其中缓存 HttpClient 实现可以作为缺省无缓存 HttpClient 实现的替代；完全可以从缓存中满足的请求不会产生实际的原始请求。旧的缓存条目在可能的情况下使用条件 GET 和 If-Modified-Since 和/或 If-None-Match 请求头自动验证原始数据。

HTTP/1.1 caching in general is designed to be emantically transparent; that is, a cache should not change the meaning of the request-response exchange between client and server. As such, it should be safe to drop a caching HttpClient into an existing compliant client-server relationship. Although the caching module is part of the client from an HTTP protocol point of view, the implementation aims to be compatible with the requirements placed on a transparent caching proxy.

一般来说，HTTP/1.1 缓存的设计是透明的；也就是说，缓存不应该改变客户机和服务器之间的请求-响应交换的含义。因此，将缓存 HttpClient 放入现有的兼容客户端-服务器关系应该是安全的。虽然从 HTTP 协议的角度来看，缓存模块是客户机的一部分，但是实现的目标是与透明缓存代理上的要求兼容。

Finally, caching HttpClient includes support the Cache-Control extensions specified by RFC 5861 (stale-if-error and stale-while-revalidate).

最后，缓存 HttpClient 包括支持 RFC 5861 指定的缓存控制扩展（stale-if-error 和 stale-while-revalidate）。

When caching HttpClient executes a request, it goes through the following flow:

当缓存 HttpClient 执行一个请求时，它经过以下流程：

- Check the request for basic compliance with the HTTP 1.1 protocol and attempt to correct the request.

检查请求是否基本符合 HTTP 1.1 协议，并尝试纠正请求。

- Flush any cache entries which would be invalidated by this request.

刷新任何缓存项，这将是无效的请求。

- Determine if the current request would be servable from cache. If not, directly pass through the request to the origin server and return the response, after caching it if appropriate.

确定当前请求是否可以从缓存中提供。如果没有，则直接将请求传递到原始服务器并返回响应，然后在适当时缓存它。

- If it was a a cache-servable request, it will attempt to read it from the cache. If it is not in the cache, call the origin server and cache the response, if appropriate.

如果它是一个 cache-servable 请求，它将尝试从缓存中读取它。如果它不在缓存中，则调用原始服务器并缓存响应（如果合适）。

- If the cached response is suitable to be served as a response, construct a BasicHttpResponse containing a ByteArrayEntity and return it. Otherwise, attempt to revalidate the cache entry against the origin server.

如果缓存的响应适合作为响应，构造一个包含 ByteArrayEntity 的 BasicHttpResponse 并返回它。否则，请尝试针对原始服务器重新验证缓存条目。

- In the case of a cached response which cannot be revalidated, call the origin server and cache the response, if appropriate.

如果缓存的响应无法重新验证，调用原始服务器并缓存响应（如果合适）。

When caching HttpClient receives a response, it goes through the following flow:

当缓存 HttpClient 接收到一个响应时，它经过以下流程：

- Examining the response for protocol compliance

检查协议遵从性的响应

- Determine whether the response is cacheable

确定响应是否可缓存

- If it is cacheable, attempt to read up to the maximum size allowed in the configuration and store it in the cache.

如果它是可缓存的，尝试读到配置中允许的最大大小，并将其存储在缓存中。

- If the response is too large for the cache, reconstruct the partially consumed response and return it directly without caching it.

如果响应对于缓存来说太大，则重构部分使用的响应直接返回，而不缓存。

It is important to note that caching HttpClient is not, itself, a different implementation of HttpClient, but that it works by inserting itself as an additonal processing component to the request execution pipeline.

需要注意的是，缓存 HttpClient 本身并不是 HttpClient 的另一种实现，而是通过将自身作为附加处理组件插入请求执行管道来实现的。

## 6.2. RFC-2616 Compliance

We believe HttpClient Cache is unconditionally compliant with RFC-2616. That is, wherever the specification indicates MUST, MUST NOT, SHOULD, or SHOULD NOT for HTTP caches, the caching layer attempts to behave in a way that satisfies those requirements. This means the caching module won't produce incorrect behavior when you drop it in.

我们相信 HttpClient 缓存是无条件兼容 RFC-2616 的。也就是说，当规范指出 HTTP 缓存 MUST、MUST NOT、SHOULD 或 SHOULD NOT 时，缓存层尝试以满足这些需求的方式运行。这意味着当您将缓存模块放入时，它不会产生不正确的行为。

## 6.3. Example Usage

This is a simple example of how to set up a basic caching HttpClient. As configured, it will store a maximum of 1000 cached objects, each of which may have a maximum body size of 8192 bytes. The numbers selected here are for example only and not intended to be prescriptive or considered as recommendations.

这是一个如何设置基本缓存 HttpClient 的简单示例。按照配置，它将存储最多 1000 个缓存对象，每个对象的最大实体大小为 8192 字节。这里选择的数字只是作为示例，并不作为规定或建议。

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

## 6.4. Configuration

The caching HttpClient inherits all configuration options and parameters of the default non-caching implementation (this includes setting options like timeouts and connection pool sizes). For caching-specific configuration, you can provide a CacheConfig instance to customize behavior across the following areas:

缓存 HttpClient 继承默认非缓存实现的所有配置选项和参数（这包括设置超时和连接池大小等选项）。对于特定于缓存的配置，您可以提供 CacheConfig 实例来定制跨以下区域的行为:

**Cache size**. If the backend storage supports these limits, you can specify the maximum number of cache entries as well as the maximum cacheable response body size.

缓存大小。如果后端存储支持这些限制，则可以指定缓存条目的最大数量以及最大的可缓存响应体大小。

**Public/private caching**. By default, the caching module considers itself to be a shared (public) cache, and will not, for example, cache responses to requests with Authorization headers or responses marked with "Cache-Control: private". If, however, the cache is only going to be used by one logical "user" (behaving similarly to a browser cache), then you will want to turn off the shared cache setting.

公共或私有缓存。默认情况下，缓存模块认为自己是一个共享的（公共的）缓存，例如，不会缓存对带有授权头或标有 Cache-Control: private 的响应的请求的响应。但是，如果缓存只由一个逻辑 user 使用（其行为类似于浏览器缓存），则需要关闭共享缓存设置。

**Heuristic caching**.Per RFC2616, a cache MAY cache certain cache entries even if no explicit cache control headers are set by the origin. This behavior is off by default, but you may want to turn this on if you are working with an origin that doesn't set proper headers but where you still want to cache the responses. You will want to enable heuristic caching, then specify either a default freshness lifetime and/or a fraction of the time since the resource was last modified. See Sections 13.2.2 and 13.2.4 of the HTTP/1.1 RFC for more details on heuristic caching.

启发式缓存。根据 RFC2616，缓存可以缓存某些缓存条目，即使原始版本没有设置显式的缓存控制头。默认情况下，此行为是关闭的，但如果您使用的原点没有设置正确的标头，但是您仍然希望在其中缓存响应，则可能需要打开此功能。您将希望启用启发式缓存，然后指定默认的新鲜度生存期和/或自上次修改资源以来的一小部分时间。有关启发式缓存的详细信息，请参阅 HTTP/1.1 RFC 的 13.2.2 和 13.2.4 节。

**Background validation**. The cache module supports the stale-while-revalidate directive of RFC5861, which allows certain cache entry revalidations to happen in the background. You may want to tweak the settings for the minimum and maximum number of background worker threads, as well as the maximum time they can be idle before being reclaimed. You can also control the size of the queue used for revalidations when there aren't enough workers to keep up with demand.

背景验证。缓存模块支持 RFC5861 的跟踪-同时-重新验证指令，它允许某些缓存条目在后台重新验证。您可能想要调整后台工作线程的最小和最大数量的设置，以及它们在被回收之前可以空闲的最大时间。您还可以控制用于重新验证的队列的大小，当没有足够的工作人员来满足需求时。

## 6.5. Storage Backends

The default implementation of caching HttpClient stores cache entries and cached response bodies in memory in the JVM of your application. While this offers high performance, it may not be appropriate for your application due to the limitation on size or because the cache entries are ephemeral and don't survive an application restart. The current release includes support for storing cache entries using EhCache and memcached implementations, which allow for spilling cache entries to disk or storing them in an external process.

缓存 HttpClient 的默认实现将缓存条目和缓存的响应主体存储在应用程序的 JVM 内存中。虽然这提供了高性能，但由于大小的限制，或者由于缓存项是临时的，无法在应用程序重启后继续存在，因此可能不适合您的应用程序。当前版本支持使用 EhCache 和 memcached 实现存储缓存条目，允许将缓存条目溢出到磁盘或将它们存储在外部进程中。

If none of those options are suitable for your application, it is possible to provide your own storage backend by implementing the HttpCacheStorage interface and then supplying that to caching HttpClient at construction time. In this case, the cache entries will be stored using your scheme but you will get to reuse all of the logic surrounding HTTP/1.1 compliance and cache handling. Generally speaking, it should be possible to create an HttpCacheStorage implementation out of anything that supports a key/value store (similar to the Java Map interface) with the ability to apply atomic updates.

如果这些选项都不适合您的应用程序，则可以通过实现 HttpCacheStorage 接口来提供自己的存储后端，然后在构建时将其提供给缓存 HttpClient。在这种情况下，缓存条目将使用您的方案存储，但是您将重用围绕 HTTP/1.1 遵从性和缓存处理的所有逻辑。一般来说，应该可以用任何支持键/值存储（类似于 Java Map 接口）并能够应用原子更新的东西来创建 HttpCacheStorage 实现。

Finally, with some extra efforts it's entirely possible to set up a multi-tier caching hierarchy; for example, wrapping an in-memory caching HttpClient around one that stores cache entries on disk or remotely in memcached, following a pattern similar to virtual memory, L1/L2 processor caches, etc.

最后，通过一些额外的努力，完全可以建立一个多层缓存层次结构；例如，按照类似于虚拟内存、L1/L2 处理器缓存等的模式，在将缓存项存储在磁盘上或远程存储在 memcached 的缓存器周围包装一个内存缓存 HttpClient。
