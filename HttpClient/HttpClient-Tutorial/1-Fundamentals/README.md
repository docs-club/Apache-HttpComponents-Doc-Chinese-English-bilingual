## Chapter 1. Fundamentals（基本原理）

### 1.1 Request execution（请求的执行）
The most essential function of HttpClient is to execute HTTP methods. Execution of an HTTP method involves one or several HTTP request / HTTP response exchanges, usually handled internally by HttpClient. The user is expected to provide a request object to execute and HttpClient is expected to transmit the request to the target server return a corresponding response object, or throw an exception if execution was unsuccessful.

HttpClient 最基本的功能是执行 HTTP 方法。HTTP 方法的执行涉及一个或多个 HTTP 请求 / HTTP 响应的交互，这通常由 HttpClient 在内部处理。用户需要提供一个要执行的请求对象，HttpClient 将请求传输到目标服务器，返回相应的响应对象，如果执行不成功，则抛出异常。

Quite naturally, the main entry point of the HttpClient API is the HttpClient interface that defines the contract described above.

显而易见，HttpClient API 的主要入口点是 HttpClient 接口，它定义了上面描述的约定。

Here is an example of request execution process in its simplest form:

下面这个例子，演示了请求的最简执行过程：

```
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/");
CloseableHttpResponse response = httpclient.execute(httpget);
try {
    <...>
} finally {
    response.close();
}
```

### 1.1.1 HTTP request（HTTP 请求）
All HTTP requests have a request line consisting a method name, a request URI and an HTTP protocol version.

所有 HTTP 请求都有一个请求行，其中包含方法名、请求 URI 和 HTTP 协议版本。

HttpClient supports out of the box all HTTP methods defined in the HTTP/1.1 specification: GET, HEAD, POST, PUT, DELETE, TRACE and OPTIONS. There is a specific class for each method type.: HttpGet, HttpHead, HttpPost, HttpPut, HttpDelete, HttpTrace, and HttpOptions.

HttpClient 支持 HTTP/1.1 规范中定义的所有开箱即用 HTTP 方法：GET、HEAD、POST、PUT、DELETE、TRACE 和 OPTIONS。每个方法类型都有一个特定的类：HttpGet、HttpHead、HttpPost、HttpPut、HttpDelete、HttpTrace 和 HttpOptions。

The Request-URI is a Uniform Resource Identifier that identifies the resource upon which to apply the request. HTTP request URIs consist of a protocol scheme, host name, optional port, resource path, optional query, and optional fragment.

Request-URI 是一个统一的资源标识符，它标识要应用请求的资源。HTTP Request-URI 由协议方案、主机名、可选端口、资源路径、可选查询和可选片段组成。

```
HttpGet httpget = new HttpGet(
     "http://www.google.com/search?hl=en&q=httpclient&btnG=Google+Search&aq=f&oq=");
```

HttpClient provides URIBuilder utility class to simplify creation and modification of request URIs.

HttpClient 提供了 URIBuilder 实用工具类来简化 Request-URI 的创建和修改。

```
URI uri = new URIBuilder()
        .setScheme("http")
        .setHost("www.google.com")
        .setPath("/search")
        .setParameter("q", "httpclient")
        .setParameter("btnG", "Google Search")
        .setParameter("aq", "f")
        .setParameter("oq", "")
        .build();
HttpGet httpget = new HttpGet(uri);
System.out.println(httpget.getURI());
```

stdout >

```
```
