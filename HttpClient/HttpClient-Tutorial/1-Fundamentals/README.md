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

HttpClient 支持 HTTP/1.1 规范中定义的所有 HTTP 方法：GET、HEAD、POST、PUT、DELETE、TRACE 和 OPTIONS。每个方法类型都有一个特定的类：HttpGet、HttpHead、HttpPost、HttpPut、HttpDelete、HttpTrace 和 HttpOptions。

The Request-URI is a Uniform Resource Identifier that identifies the resource upon which to apply the request. HTTP request URIs consist of a protocol scheme, host name, optional port, resource path, optional query, and optional fragment.

Request-URI 是统一的资源标识符，它标识要应用请求的资源。HTTP Request-URI 由协议方案、主机名、可选端口、资源路径、可选查询和可选片段组成。

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

输出如下结果：

```
http://www.google.com/search?q=httpclient&btnG=Google+Search&aq=f&oq=
```

### 1.1.2 HTTP response（HTTP 响应）
HTTP response is a message sent by the server back to the client after having received and interpreted a request message. The first line of that message consists of the protocol version followed by a numeric status code and its associated textual phrase.

HTTP 响应是服务器在接收并解释请求消息之后发送回客户端的消息。该消息的第一行由协议版本、数字状态代码及其相关的文本短语组成。

```
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1, HttpStatus.SC_OK, "OK");

System.out.println(response.getProtocolVersion());
System.out.println(response.getStatusLine().getStatusCode());
System.out.println(response.getStatusLine().getReasonPhrase());
System.out.println(response.getStatusLine().toString());
```

stdout >

输出如下结果：

```
HTTP/1.1
200
OK
HTTP/1.1 200 OK
```

### 1.1.3 Working with message headers（处理消息头）
An HTTP message can contain a number of headers describing properties of the message such as the content length, content type and so on. HttpClient provides methods to retrieve, add, remove and enumerate headers.

HTTP 消息可以包含许多消息头，它们描述了消息属性（如内容长度、内容类型等）。HttpClient 提供了检索、添加、删除和枚举消息头的方法。

```
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1, HttpStatus.SC_OK, "OK");
response.addHeader("Set-Cookie", "c1=a; path=/; domain=localhost");
response.addHeader("Set-Cookie", "c2=b; path=\"/\", c3=c; domain=\"localhost\"");
Header h1 = response.getFirstHeader("Set-Cookie");
System.out.println(h1);
Header h2 = response.getLastHeader("Set-Cookie");
System.out.println(h2);
Header[] hs = response.getHeaders("Set-Cookie");
System.out.println(hs.length);
```

stdout >

输出如下结果：

```
Set-Cookie: c1=a; path=/; domain=localhost
Set-Cookie: c2=b; path="/", c3=c; domain="localhost"
2
```

The most efficient way to obtain all headers of a given type is by using the HeaderIterator interface.

获取给定类型的所有消息头，最有效方法是使用 HeaderIterator 接口。

```
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1, HttpStatus.SC_OK, "OK");
response.addHeader("Set-Cookie", "c1=a; path=/; domain=localhost");
response.addHeader("Set-Cookie", "c2=b; path=\"/\", c3=c; domain=\"localhost\"");

HeaderIterator it = response.headerIterator("Set-Cookie");

while (it.hasNext()) {
    System.out.println(it.next());
}
```

stdout >

输出如下：

```
Set-Cookie: c1=a; path=/; domain=localhost
Set-Cookie: c2=b; path="/", c3=c; domain="localhost"
```

It also provides convenience methods to parse HTTP messages into individual header elements.

HttpClient 还提供了便捷的方法来将 HTTP 消息解析为单独的消息头元素。

```
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1, HttpStatus.SC_OK, "OK");
response.addHeader("Set-Cookie", "c1=a; path=/; domain=localhost");
response.addHeader("Set-Cookie", "c2=b; path=\"/\", c3=c; domain=\"localhost\"");

HeaderElementIterator it = new BasicHeaderElementIterator(response.headerIterator("Set-Cookie"));

while (it.hasNext()) {
    HeaderElement elem = it.nextElement();
    System.out.println(elem.getName() + " = " + elem.getValue());
    NameValuePair[] params = elem.getParameters();
    for (int i = 0; i < params.length; i++) {
        System.out.println(" " + params[i]);
    }
}
```

stdout >

输出如下：

```
c1 = a
path=/
domain=localhost
c2 = b
path=/
c3 = c
domain=localhost
```

### 1.1.4 HTTP entity（HTTP 实体）
HTTP messages can carry a content entity associated with the request or response. Entities can be found in some requests and in some responses, as they are optional. Requests that use entities are referred to as entity enclosing requests. The HTTP specification defines two entity enclosing request methods: `POST` and `PUT`. Responses are usually expected to enclose a content entity. There are exceptions to this rule such as responses to `HEAD` method and `204 No Content, 304 Not Modified, 205 Reset Content` responses.

HTTP 消息可以携带与请求或响应相关联的内容实体。实体可以在一些请求和响应中找到，因为它们是可选的。**使用实体的请求称为包含请求的实体。** HTTP 规范定义了两个包含请求方法的实体：`POST` 和 `PUT`。通常认为响应包含一个内容实体。这个规则也有例外，比如对 `HEAD` 方法的响应和 `204 No Content, 304 Not Modified, 205 Reset Content` 响应。

HttpClient distinguishes three kinds of entities, depending on where their content originates:

HttpClient 根据内容的来源将实体分为三种：

- streamed:  The content is received from a stream, or generated on the fly. In particular, this category includes entities being received from HTTP responses. Streamed entities are generally not repeatable.

流：内容从流中接收，或动态生成。特别地，这个类别包括从 HTTP 响应接收的实体。流实体通常是不可重复的。

- self-contained:  The content is in memory or obtained by means that are independent from a connection or other entity. Self-contained entities are generally repeatable. This type of entities will be mostly used for entity enclosing HTTP requests.

自包含的：内容在内存中，或者通过独立于连接或其他实体的方式获取。自包含实体通常是可重复的。这种类型的实体主要用于封装 HTTP 请求的实体。

**译注：此处需要例子说明。**

- wrapping:  The content is obtained from another entity.

包装：内容从另一个实体获得。

This distinction is important for connection management when streaming out content from an HTTP response. For request entities that are created by an application and only sent using HttpClient, the difference between streamed and self-contained is of little importance. In that case, it is suggested to consider non-repeatable entities as streamed, and those that are repeatable as self-contained.

当从 HTTP 响应中输出内容时，这些区别对于连接管理非常重要。对于由应用程序创建且仅使用 HttpClient 发送的请求实体，流和自包含的区别并不重要。在这种情况下，建议将不可重复的实体视为流，将可重复的实体视为自包含的。

#### 1.1.4.1 Repeatable entities（可重复的实体）
An entity can be repeatable, meaning its content can be read more than once. This is only possible with self contained entities (like `ByteArrayEntity` or `StringEntity`)

一个实体可以重复，这意味着它的内容可以被多次读取。**这只可能与自包含实体**（如 `ByteArrayEntity` 或 `StringEntity`）

#### 1.1.4.2 Using HTTP entities（使用 HTTP 实体）
Since an entity can represent both binary and character content, it has support for character encodings (to support the latter, ie. character content).

由于实体既可以表示二进制内容，也可以表示字符内容，所以它支持字符编码（**支持后者，那就是字符内容**)。

The entity is created when executing a request with enclosed content or when the request was successful and the response body is used to send the result back to the client.

实体是在执行包含内容的请求时或请求成功时创建的，响应体用于将结果发送回客户端。

To read the content from the entity, one can either retrieve the input stream via the `HttpEntity#getContent()` method, which returns an `java.io.InputStream`, or one can supply an output stream to the `HttpEntity#writeTo(OutputStream)` method, which will return once all content has been written to the given stream.

要从实体中读取内容，可以通过 `HttpEntity#getContent()` 方法，它返回 `java.io.InputStream`，也可以向 `HttpEntity#writeTo(OutputStream)` 方法提供输出流，该方法将所有内容写入给定流后返回。

When the entity has been received with an incoming message, the methods `HttpEntity#getContentType()` and `HttpEntity#getContentLength()` methods can be used for reading the common metadata such as `Content-Type` and `Content-Length` headers (if they are available). Since the `Content-Type` header can contain a character encoding for text mime-types like text/plain or text/html, the `HttpEntity#getContentEncoding()` method is used to read this information. If the headers aren't available, a length of -1 will be returned, and NULL for the content type. If the `Content-Type` header is available, a `Header` object will be returned.

当接收到带有传入消息的实体时，方法 `HttpEntity#getContentType()` 和 `HttpEntity#getContentLength()` 可用于读取公共元数据，如 `Content-Type` 和 `Content-Length` 头信息（如果它们可用）。由于 `Content-Type` 头可以包含文本的媒体类型（如 text/plain 或 text/html）的字符编码，因此使用 `HttpEntity#getContentEncoding()` 方法来读取此信息。如果头不可用，则返回长度 -1，内容类型为 NULL。如果 `Content-Type` 头可用，则返回一个 `Header` 对象。

When creating an entity for a outgoing message, this meta data has to be supplied by the creator of the entity.

在为传出消息创建实体时，必须由实体的创建者提供此元数据。

```
StringEntity myEntity = new StringEntity("important message", ContentType.create("text/plain", "UTF-8"));

System.out.println(myEntity.getContentType());
System.out.println(myEntity.getContentLength());
System.out.println(EntityUtils.toString(myEntity));
System.out.println(EntityUtils.toByteArray(myEntity).length);
```

stdout >

输出如下：

```
Content-Type: text/plain; charset=utf-8
17
important message
17
```

### 1.1.5 Ensuring release of low level resources（确保底层资源的释放）
In order to ensure proper release of system resources one must close either the content stream associated with the entity or the response itself

为了确保正确释放系统资源，必须关闭与实体或响应本身关联的内容流。

```
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/");
CloseableHttpResponse response = httpclient.execute(httpget);
try {
    HttpEntity entity = response.getEntity();
    if (entity != null) {
        InputStream instream = entity.getContent();
        try {
            // do something useful
        } finally {
            instream.close();
        }
    }
} finally {
    response.close();
}
```

The difference between closing the content stream and closing the response is that the former will attempt to keep the underlying connection alive by consuming the entity content while the latter immediately shuts down and discards the connection.

关闭内容流和关闭响应之间的区别在于，前者将尝试通过使用实体内容来保持底层连接的活动，而后者将立即关闭并丢弃连接。

Please note that the `HttpEntity#writeTo(OutputStream)` method is also required to ensure proper release of system resources once the entity has been fully written out. If this method obtains an instance of `java.io.InputStream` by calling `HttpEntity#getContent()`, it is also expected to close the stream in a finally clause.

请注意，`HttpEntity#writeTo(OutputStream)` 方法也需要确保在实体被完全写完之后，系统资源得到适当的释放。如果该方法通过调用 `HttpEntity#getContent()` 获得 `java.io.InputStream` 实例，它还将在 finally 子句中关闭流。

When working with streaming entities, one can use the `EntityUtils#consume(HttpEntity)` method to ensure that the entity content has been fully consumed and the underlying stream has been closed.

在处理流实体时，可以使用 `EntityUtils#consume(HttpEntity)` 方法来确保实体内容已被完全使用，并且底层流已被关闭。

There can be situations, however, when only a small portion of the entire response content needs to be retrieved and the performance penalty for consuming the remaining content and making the connection reusable is too high, in which case one can terminate the content stream by closing the response.

但是，当需要检索整个响应内容的一小部分时，使用剩余内容和使连接可重用的性能代价太高，这时可以通过关闭响应终止内容流。

```
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/");
CloseableHttpResponse response = httpclient.execute(httpget);
try {
    HttpEntity entity = response.getEntity();
    if (entity != null) {
        InputStream instream = entity.getContent();
        int byteOne = instream.read();
        int byteTwo = instream.read();
        // Do not need the rest
    }
} finally {
    response.close();
}
```

The connection will not be reused, but all level resources held by it will be correctly deallocated.

连接将不会被重用，但它所持有的所有层级的资源将被正确释放。

### 1.1.6 Consuming entity content
The recommended way to consume the content of an entity is by using its `HttpEntity#getContent()` or `HttpEntity#writeTo(OutputStream)` methods. HttpClient also comes with the `EntityUtils` class, which exposes several static methods to more easily read the content or information from an entity. Instead of reading the `java.io.InputStream` directly, one can retrieve the whole content body in a string / byte array by using the methods from this class. However, the use of `EntityUtils` is strongly discouraged unless the response entities originate from a trusted HTTP server and are known to be of limited length.

```
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/");
CloseableHttpResponse response = httpclient.execute(httpget);
try {
    HttpEntity entity = response.getEntity();
    if (entity != null) {
        long len = entity.getContentLength();
        if (len != -1 && len < 2048) {
            System.out.println(EntityUtils.toString(entity));
        } else {
            // Stream content out
        }
    }
} finally {
    response.close();
}
```

In some situations it may be necessary to be able to read entity content more than once. In this case entity content must be buffered in some way, either in memory or on disk. The simplest way to accomplish that is by wrapping the original entity with the `BufferedHttpEntity` class. This will cause the content of the original entity to be read into a in-memory buffer. In all other ways the entity wrapper will be have the original one.

```
CloseableHttpResponse response = <...>
HttpEntity entity = response.getEntity();
if (entity != null) {
    entity = new BufferedHttpEntity(entity);
}
```

### 1.1.7 Producing entity content
HttpClient provides several classes that can be used to efficiently stream out content throught HTTP connections. Instances of those classes can be associated with entity enclosing requests such as `POST` and `PUT` in order to enclose entity content into outgoing HTTP requests. HttpClient provides several classes for most common data containers such as string, byte array, input stream, and file: `StringEntity`, `ByteArrayEntity`, `InputStreamEntity`, and `FileEntity`.

```
File file = new File("somefile.txt");
FileEntity entity = new FileEntity(file, ContentType.create("text/plain", "UTF-8"));        

HttpPost httppost = new HttpPost("http://localhost/action.do");
httppost.setEntity(entity);
```

Please note `InputStreamEntity` is not repeatable, because it can only read from the underlying data stream once. Generally it is recommended to implement a custom `HttpEntity` class which is self-contained instead of using the generic `InputStreamEntity.FileEntity` can be a good starting point.

#### 1.1.7.1 HTML forms
Many applications need to simulate the process of submitting an HTML form, for instance, in order to log in to a web application or submit input data. HttpClient provides the entity class `UrlEncodedFormEntity` to facilitate the process.

```
List<NameValuePair> formparams = new ArrayList<NameValuePair>();
formparams.add(new BasicNameValuePair("param1", "value1"));
formparams.add(new BasicNameValuePair("param2", "value2"));
UrlEncodedFormEntity entity = new UrlEncodedFormEntity(formparams, Consts.UTF_8);
HttpPost httppost = new HttpPost("http://localhost/handler.do");
httppost.setEntity(entity);
```

The `UrlEncodedFormEntity` instance will use the so called URL encoding to encode parameters and produce the following content:

```
param1=value1&param2=value2
```

#### 1.1.7.2. Content chunking
Generally it is recommended to let HttpClient choose the most appropriate transfer encoding based on the properties of the HTTP message being transferred. It is possible, however, to inform HttpClient that chunk coding is preferred by setting `HttpEntity#setChunked()` to true. Please note that HttpClient will use this flag as a hint only. This value will be ignored when using HTTP protocol versions that do not support chunk coding, such as HTTP/1.0.

```
StringEntity entity = new StringEntity("important message",
        ContentType.create("plain/text", Consts.UTF_8));
entity.setChunked(true);
HttpPost httppost = new HttpPost("http://localhost/acrtion.do");
httppost.setEntity(entity);
```

#### 1.1.8. Response handlers
The simplest and the most convenient way to handle responses is by using the `ResponseHandler` interface, which includes the `handleResponse(HttpResponse response)` method. This method completely relieves the user from having to worry about connection management. When using a `ResponseHandler`, HttpClient will automatically take care of ensuring release of the connection back to the connection manager regardless whether the request execution succeeds or causes an exception.

```
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/json");

ResponseHandler<MyJsonObject> rh = new ResponseHandler<MyJsonObject>() {

    @Override
    public JsonObject handleResponse(
            final HttpResponse response) throws IOException {
        StatusLine statusLine = response.getStatusLine();
        HttpEntity entity = response.getEntity();
        if (statusLine.getStatusCode() >= 300) {
            throw new HttpResponseException(
                    statusLine.getStatusCode(),
                    statusLine.getReasonPhrase());
        }
        if (entity == null) {
            throw new ClientProtocolException("Response contains no content");
        }
        Gson gson = new GsonBuilder().create();
        ContentType contentType = ContentType.getOrDefault(entity);
        Charset charset = contentType.getCharset();
        Reader reader = new InputStreamReader(entity.getContent(), charset);
        return gson.fromJson(reader, MyJsonObject.class);
    }
};
MyJsonObject myjson = client.execute(httpget, rh);
```

## 1.2. HttpClient interface
`HttpClient` interface represents the most essential contract for HTTP request execution. It imposes no restrictions or particular details on the request execution process and leaves the specifics of connection management, state management, authentication and redirect handling up to individual implementations. This should make it easier to decorate the interface with additional functionality such as response content caching.

Generally `HttpClient` implementations act as a facade to a number of special purpose handler or strategy interface implementations responsible for handling of a particular aspect of the HTTP protocol such as redirect or authentication handling or making decision about connection persistence and keep alive duration. This enables the users to selectively replace default implementation of those aspects with custom, application specific ones.

```
ConnectionKeepAliveStrategy keepAliveStrat = new DefaultConnectionKeepAliveStrategy() {

    @Override
    public long getKeepAliveDuration(
            HttpResponse response,
            HttpContext context) {
        long keepAlive = super.getKeepAliveDuration(response, context);
        if (keepAlive == -1) {
            // Keep connections alive 5 seconds if a keep-alive value
            // has not be explicitly set by the server
            keepAlive = 5000;
        }
        return keepAlive;
    }

};
CloseableHttpClient httpclient = HttpClients.custom()
        .setKeepAliveStrategy(keepAliveStrat)
        .build();
```

### 1.2.1. HttpClient thread safety
`HttpClient` implementations are expected to be thread safe. It is recommended that the same instance of this class is reused for multiple request executions.

### 1.2.2. HttpClient resource deallocation
When an instance CloseableHttpClient is no longer needed and is about to go out of scope the connection manager associated with it must be shut down by calling the `CloseableHttpClient#close()` method.

```
CloseableHttpClient httpclient = HttpClients.createDefault();
try {
    <...>
} finally {
    httpclient.close();
}
```

## 1.3. HTTP execution context
Originally HTTP has been designed as a stateless, response-request oriented protocol. However, real world applications often need to be able to persist state information through several logically related request-response exchanges. In order to enable applications to maintain a processing state HttpClient allows HTTP requests to be executed within a particular execution context, referred to as HTTP context. Multiple logically related requests can participate in a logical session if the same context is reused between consecutive requests. HTTP context functions similarly to a `java.util.Map<String, Object>`. It is simply a collection of arbitrary named values. An application can populate context attributes prior to request execution or examine the context after the execution has been completed.

`HttpContext` can contain arbitrary objects and therefore may be unsafe to share between multiple threads. It is recommended that each thread of execution maintains its own context.

In the course of HTTP request execution HttpClient adds the following attributes to the execution context:

- `HttpConnection` instance representing the actual connection to the target server.
- `HttpHost` instance representing the connection target.
- `HttpRoute` instance representing the complete connection route
- `HttpRequest` instance representing the actual HTTP request. The final HttpRequest object in the execution context always represents the state of the message exactly as it was sent to the target server. Per default HTTP/1.0 and HTTP/1.1 use relative request URIs. However if the request is sent via a proxy in a non-tunneling mode then the URI will be absolute.
- `HttpResponse` instance representing the actual HTTP response.
- `java.lang.Boolean` object representing the flag indicating whether the actual request has been fully transmitted to the connection target.
- `RequestConfig` object representing the actual request configuation.
- `java.util.List<URI>` object representing a collection of all redirect locations received in the process of request execution.

One can use HttpClientContext adaptor class to simplify interractions with the context state.

```
HttpContext context = <...>
HttpClientContext clientContext = HttpClientContext.adapt(context);
HttpHost target = clientContext.getTargetHost();
HttpRequest request = clientContext.getRequest();
HttpResponse response = clientContext.getResponse();
RequestConfig config = clientContext.getRequestConfig();
```

Multiple request sequences that represent a logically related session should be executed with the same `HttpContext` instance to ensure automatic propagation of conversation context and state information between requests.

In the following example the request configuration set by the initial request will be kept in the execution context and get propagated to the consecutive requests sharing the same context.

```
CloseableHttpClient httpclient = HttpClients.createDefault();
RequestConfig requestConfig = RequestConfig.custom()
        .setSocketTimeout(1000)
        .setConnectTimeout(1000)
        .build();

HttpGet httpget1 = new HttpGet("http://localhost/1");
httpget1.setConfig(requestConfig);
CloseableHttpResponse response1 = httpclient.execute(httpget1, context);
try {
    HttpEntity entity1 = response1.getEntity();
} finally {
    response1.close();
}
HttpGet httpget2 = new HttpGet("http://localhost/2");
CloseableHttpResponse response2 = httpclient.execute(httpget2, context);
try {
    HttpEntity entity2 = response2.getEntity();
} finally {
    response2.close();
}
```

## 1.4. HTTP protocol interceptors
The HTTP protocol interceptor is a routine that implements a specific aspect of the HTTP protocol. Usually protocol interceptors are expected to act upon one specific header or a group of related headers of the incoming message, or populate the outgoing message with one specific header or a group of related headers. Protocol interceptors can also manipulate content entities enclosed with messages - transparent content compression / decompression being a good example. Usually this is accomplished by using the 'Decorator' pattern where a wrapper entity class is used to decorate the original entity. Several protocol interceptors can be combined to form one logical unit.

Protocol interceptors can collaborate by sharing information - such as a processing state - through the HTTP execution context. Protocol interceptors can use HTTP context to store a processing state for one request or several consecutive requests.

Usually the order in which interceptors are executed should not matter as long as they do not depend on a particular state of the execution context. If protocol interceptors have interdependencies and therefore must be executed in a particular order, they should be added to the protocol processor in the same sequence as their expected execution order.

Protocol interceptors must be implemented as thread-safe. Similarly to servlets, protocol interceptors should not use instance variables unless access to those variables is synchronized.

This is an example of how local context can be used to persist a processing state between consecutive requests:

```
CloseableHttpClient httpclient = HttpClients.custom()
        .addInterceptorLast(new HttpRequestInterceptor() {

            public void process(
                    final HttpRequest request,
                    final HttpContext context) throws HttpException, IOException {
                AtomicInteger count = (AtomicInteger) context.getAttribute("count");
                request.addHeader("Count", Integer.toString(count.getAndIncrement()));
            }

        })
        .build();

AtomicInteger count = new AtomicInteger(1);
HttpClientContext localContext = HttpClientContext.create();
localContext.setAttribute("count", count);

HttpGet httpget = new HttpGet("http://localhost/");
for (int i = 0; i < 10; i++) {
    CloseableHttpResponse response = httpclient.execute(httpget, localContext);
    try {
        HttpEntity entity = response.getEntity();
    } finally {
        response.close();
    }
}
```

## 1.5. Exception handling
HTTP protocol processors can throw two types of exceptions: `java.io.IOException` in case of an I/O failure such as socket timeout or an socket reset and `HttpException` that signals an HTTP failure such as a violation of the HTTP protocol. Usually I/O errors are considered non-fatal and recoverable, whereas HTTP protocol errors are considered fatal and cannot be automatically recovered from. Please note that `HttpClient` implementations re-throw `HttpExceptions` as `ClientProtocolException`, which is a subclass of `java.io.IOException`. This enables the users of `HttpClient` to handle both I/O errors and protocol violations from a single catch clause.

### 1.5.1. HTTP transport safety
It is important to understand that the HTTP protocol is not well suited to all types of applications. HTTP is a simple request/response oriented protocol which was initially designed to support static or dynamically generated content retrieval. It has never been intended to support transactional operations. For instance, the HTTP server will consider its part of the contract fulfilled if it succeeds in receiving and processing the request, generating a response and sending a status code back to the client. The server will make no attempt to roll back the transaction if the client fails to receive the response in its entirety due to a read timeout, a request cancellation or a system crash. If the client decides to retry the same request, the server will inevitably end up executing the same transaction more than once. In some cases this may lead to application data corruption or inconsistent application state.

Even though HTTP has never been designed to support transactional processing, it can still be used as a transport protocol for mission critical applications provided certain conditions are met. To ensure HTTP transport layer safety the system must ensure the idempotency of HTTP methods on the application layer.

### 1.5.2. Idempotent methods
HTTP/1.1 specification defines an idempotent method as

[Methods can also have the property of "idempotence" in that (aside from error or expiration issues) the side-effects of N > 0 identical requests is the same as for a single request]

In other words the application ought to ensure that it is prepared to deal with the implications of multiple execution of the same method. This can be achieved, for instance, by providing a unique transaction id and by other means of avoiding execution of the same logical operation.

Please note that this problem is not specific to HttpClient. Browser based applications are subject to exactly the same issues related to HTTP methods non-idempotency.

By default HttpClient assumes only non-entity enclosing methods such as `GET` and `HEAD` to be idempotent and entity enclosing methods such as POST and PUT to be not for compatibility reasons.

### 1.5.3. Automatic exception recovery
By default HttpClient attempts to automatically recover from I/O exceptions. The default auto-recovery mechanism is limited to just a few exceptions that are known to be safe.

- HttpClient will make no attempt to recover from any logical or HTTP protocol errors (those derived from `HttpException` class).
- HttpClient will automatically retry those methods that are assumed to be idempotent.
- HttpClient will automatically retry those methods that fail with a transport exception while the HTTP request is still being transmitted to the target server (i.e. the request has not been fully transmitted to the server).

### 1.5.4. Request retry handler
In order to enable a custom exception recovery mechanism one should provide an implementation of the HttpRequestRetryHandler interface.

```
HttpRequestRetryHandler myRetryHandler = new HttpRequestRetryHandler() {

    public boolean retryRequest(
            IOException exception,
            int executionCount,
            HttpContext context) {
        if (executionCount >= 5) {
            // Do not retry if over max retry count
            return false;
        }
        if (exception instanceof InterruptedIOException) {
            // Timeout
            return false;
        }
        if (exception instanceof UnknownHostException) {
            // Unknown host
            return false;
        }
        if (exception instanceof ConnectTimeoutException) {
            // Connection refused
            return false;
        }
        if (exception instanceof SSLException) {
            // SSL handshake exception
            return false;
        }
        HttpClientContext clientContext = HttpClientContext.adapt(context);
        HttpRequest request = clientContext.getRequest();
        boolean idempotent = !(request instanceof HttpEntityEnclosingRequest);
        if (idempotent) {
            // Retry if the request is considered idempotent
            return true;
        }
        return false;
    }

};
CloseableHttpClient httpclient = HttpClients.custom()
        .setRetryHandler(myRetryHandler)
        .build();
```

Please note that one can use StandardHttpRequestRetryHandler instead of the one used by default in order to treat those request methods defined as idempotent by RFC-2616 as safe to retry automatically: GET, HEAD, PUT, DELETE, OPTIONS, and TRACE.

## 1.6. Aborting requests
In some situations HTTP request execution fails to complete within the expected time frame due to high load on the target server or too many concurrent requests issued on the client side. In such cases it may be necessary to terminate the request prematurely and unblock the execution thread blocked in a I/O operation. HTTP requests being executed by HttpClient can be aborted at any stage of execution by invoking `HttpUriRequest#abort()` method. This method is thread-safe and can be called from any thread. When an HTTP request is aborted its execution thread - even if currently blocked in an I/O operation - is guaranteed to unblock by throwing a `InterruptedIOException`

## 1.7. Redirect handling
HttpClient handles all types of redirects automatically, except those explicitly prohibited by the HTTP specification as requiring user intervention. `See Other` (status code 303) redirects on `POST` and `PUT` requests are converted to `GET` requests as required by the HTTP specification. One can use a custom redirect strategy to relaxe restrictions on automatic redirection of POST methods imposed by the HTTP specification.

```
LaxRedirectStrategy redirectStrategy = new LaxRedirectStrategy();
CloseableHttpClient httpclient = HttpClients.custom()
        .setRedirectStrategy(redirectStrategy)
        .build();
```

HttpClient often has to rewrite the request message in the process of its execution. Per default HTTP/1.0 and HTTP/1.1 generally use relative request URIs. Likewise, original request may get redirected from location to another multiple times. The final interpreted absolute HTTP location can be built using the original request and the context. The utility method `URIUtils#resolve` can be used to build the interpreted absolute URI used to generate the final request. This method includes the last fragment identifier from the redirect requests or the original request.

```
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpClientContext context = HttpClientContext.create();
HttpGet httpget = new HttpGet("http://localhost:8080/");
CloseableHttpResponse response = httpclient.execute(httpget, context);
try {
    HttpHost target = context.getTargetHost();
    List<URI> redirectLocations = context.getRedirectLocations();
    URI location = URIUtils.resolve(httpget.getURI(), target, redirectLocations);
    System.out.println("Final HTTP location: " + location.toASCIIString());
    // Expected to be an absolute URI
} finally {
    response.close();
}
```
