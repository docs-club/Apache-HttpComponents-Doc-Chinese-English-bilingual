# Chapter 2. Blocking I/O model

Blocking (or classic) I/O in Java represents a highly efficient and convenient I/O model well suited for high performance applications where the number of concurrent connections is relatively moderate. Modern JVMs are capable of efficient context switching and the blocking I/O model should offer the best performance in terms of raw data throughput as long as the number of concurrent connections is below one thousand and connections are mostly busy transmitting data. However for applications where connections stay idle most of the time the overhead of context switching may become substantial and a non-blocking I/O model may present a better alternative.

Java 中的阻塞（或经典）I/O 代表了一种高效、方便的 I/O 模型，非常适合并发连接数量相对适中的高性能应用程序。现代 jvm 能够进行有效的上下文切换，而阻塞 I/O 模型应该能够提供原始数据吞吐量方面的最佳性能，只要并发连接的数量低于 1000，并且连接主要忙于传输数据。然而，对于连接大部分时间处于空闲状态的应用程序，上下文切换的开销可能会变得很大，而非阻塞 I/O 模型提供了更好的选择。

## 2.1. Blocking HTTP connections

HTTP connections are responsible for HTTP message serialization and deserialization. One should rarely need to use HTTP connection objects directly. There are higher level protocol components intended for execution and processing of HTTP requests. However, in some cases direct interaction with HTTP connections may be necessary, for instance, to access properties such as the connection status, the socket timeout or the local and remote addresses.

HTTP 连接负责 HTTP 消息的序列化和反序列化。很少需要去直接使用 HTTP 连接对象。还有更高级别的协议组件用于执行和处理 HTTP 请求。但是，在某些情况下，可能需要与 HTTP 连接进行直接交互，例如访问连接状态、套接字超时或本地和远程地址等属性。

It is important to bear in mind that HTTP connections are not thread-safe. We strongly recommend limiting all interactions with HTTP connection objects to one thread. The only method of HttpConnection interface and its sub-interfaces which is safe to invoke from another thread is HttpConnection#shutdown() .

重要的是要记住 HTTP 连接不是线程安全的。我们强烈建议将所有与 HTTP 连接对象的交互限制在一个线程内。从另一个线程调用 HttpConnection 接口及其子接口的惟一安全方法是 HttpConnection#shutdown()。

### 2.1.1. Working with blocking HTTP connections

HttpCore does not provide full support for opening connections because the process of establishing a new connection - especially on the client side - can be very complex when it involves one or more authenticating or/and tunneling proxies. Instead, blocking HTTP connections can be bound to any arbitrary network socket.

HttpCore 并不完全支持打开连接，因为建立新连接的过程（尤其是在客户端）可能非常复杂，因为它涉及一个或多个身份验证或/和隧道代理。相反，阻塞 HTTP 连接可以绑定到任意网络套接字。

```
Socket socket = <...>

DefaultBHttpClientConnection conn = new DefaultBHttpClientConnection(8 * 1024);
conn.bind(socket);
System.out.println(conn.isOpen());
HttpConnectionMetrics metrics = conn.getMetrics();
System.out.println(metrics.getRequestCount());
System.out.println(metrics.getResponseCount());
System.out.println(metrics.getReceivedBytesCount());
System.out.println(metrics.getSentBytesCount());
```

HTTP connection interfaces, both client and server, send and receive messages in two stages. The message head is transmitted first. Depending on properties of the message head, a message body may follow it. Please note it is very important to always close the underlying content stream in order to signal that the processing of the message is complete. HTTP entities that stream out their content directly from the input stream of the underlying connection must ensure they fully consume the content of the message body for that connection to be potentially re-usable.

HTTP 连接接口（客户端和服务器）分两个阶段发送和接收消息。首先传输消息头。根据消息头的属性，消息正文可以跟随它。请注意，关闭底层内容流是非常重要的，表明消息处理已经完成。直接从底层连接的输入流中输出内容的 HTTP 实体必须确保它们完全使用消息体的内容，以便该连接具有潜在的可重用性。

Over-simplified process of request execution on the client side may look like this:

简化后的请求执行过程在客户端可能是这样的：

```
Socket socket = <...>

DefaultBHttpClientConnection conn = new DefaultBHttpClientConnection(8 * 1024);
conn.bind(socket);
HttpRequest request = new BasicHttpRequest("GET", "/");
conn.sendRequestHeader(request);
HttpResponse response = conn.receiveResponseHeader();
conn.receiveResponseEntity(response);
HttpEntity entity = response.getEntity();
if (entity != null) {
    // Do something useful with the entity and, when done, ensure all
    // content has been consumed, so that the underlying connection
    // can be re-used
    EntityUtils.consume(entity);
}
```

Over-simplified process of request handling on the server side may look like this:

服务器端简化后的请求处理过程可能是这样的：

```
Socket socket = <...>

DefaultBHttpServerConnection conn = new DefaultBHttpServerConnection(8 * 1024);
conn.bind(socket);
HttpRequest request = conn.receiveRequestHeader();
if (request instanceof HttpEntityEnclosingRequest) {
    conn.receiveRequestEntity((HttpEntityEnclosingRequest) request);
    HttpEntity entity = ((HttpEntityEnclosingRequest) request)
            .getEntity();
    if (entity != null) {
        // Do something useful with the entity and, when done, ensure all
        // content has been consumed, so that the underlying connection
        // could be re-used
        EntityUtils.consume(entity);
    }
}
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1,
        200, "OK") ;
response.setEntity(new StringEntity("Got it") );
conn.sendResponseHeader(response);
conn.sendResponseEntity(response);
```

Please note that one should rarely need to transmit messages using these low level methods and should normally use the appropriate higher level HTTP service implementations instead.

请注意，很少需要使用这些低级方法来传输消息，通常应该使用适当的高级 HTTP 服务实现。

### 2.1.2. Content transfer with blocking I/O

HTTP connections manage the process of the content transfer using the HttpEntity interface. HTTP connections generate an entity object that encapsulates the content stream of the incoming message. Please note that HttpServerConnection#receiveRequestEntity() and HttpClientConnection#receiveResponseEntity() do not retrieve or buffer any incoming data. They merely inject an appropriate content codec based on the properties of the incoming message. The content can be retrieved by reading from the content input stream of the enclosed entity using HttpEntity#getContent(). The incoming data will be decoded automatically and completely transparently to the data consumer. Likewise, HTTP connections rely on HttpEntity#writeTo(OutputStream) method to generate the content of an outgoing message. If an outgoing message encloses an entity, the content will be encoded automatically based on the properties of the message.

HTTP 连接使用 HttpEntity 接口管理内容传输的过程。HTTP 连接生成一个实体对象，该对象封装传入消息的内容流。请注意 HttpServerConnection#receiveRequestEntity() 和 HttpClientConnection#receiveResponseEntity() 不检索或缓冲任何传入数据。它们只是根据传入消息的属性注入适当的内容编解码器。可以使用 HttpEntity#getContent() 从所包含的实体的内容输入流中读取内容。传入的数据将被自动解码，并且对数据使用者完全透明。同样，HTTP 连接依赖于 HttpEntity#writeTo(OutputStream) 方法来生成传出消息的内容。如果传出消息包含一个实体，则根据消息的属性自动对内容进行编码。

### 2.1.3. Supported content transfer mechanisms

Default implementations of HTTP connections support three content transfer mechanisms defined by the HTTP/1.1 specification:

HTTP 连接的默认实现支持 HTTP/1.1 规范定义的三种内容传输机制：

- Content-Length delimited: The end of the content entity is determined by the value of the Content-Length header. Maximum entity length: Long#MAX_VALUE.

内容长度分隔：内容实体的结尾由内容长度头部的值决定。最大实体长度：Long#MAX_VALUE

- Identity coding: The end of the content entity is demarcated by closing the underlying connection (end of stream condition). For obvious reasons the identity encoding can only be used on the server side. Maximum entity length: unlimited.

标识编码：通过关闭底层连接（流的结束条件）来划分内容实体的结束。由于明显的原因，标识编码只能在服务器端使用。最大实体长度：无限。

- Chunk coding: The content is sent in small chunks. Maximum entity length: unlimited.

Chunk 编码：内容以小块的形式发送。最大实体长度：无限。

The appropriate content stream class will be created automatically depending on properties of the entity enclosed with the message.

适当的内容流类将根据消息所包含的实体的属性自动创建。

### 2.1.4. Terminating HTTP connections

HTTP connections can be terminated either gracefully by calling HttpConnection#close() or forcibly by calling HttpConnection#shutdown(). The former tries to flush all buffered data prior to terminating the connection and may block indefinitely. The HttpConnection#close() method is not thread-safe. The latter terminates the connection without flushing internal buffers and returns control to the caller as soon as possible without blocking for long. The HttpConnection#shutdown() method is thread-safe.

HTTP 连接可以通过调用 HttpConnection#close() 来优雅地终止，也可以通过调用 HttpConnection#shutdown() 强制终止。前者试图在终止连接之前刷新所有缓冲的数据，可能会无限期阻塞。HttpConnection#close() 方法不是线程安全的。后者在不刷新内部缓冲区的情况下终止连接，并在不长时间阻塞的情况下尽快将控制权返回给调用者。HttpConnection#shutdown() 方法是线程安全的。

## 2.2. HTTP exception handling

All HttpCore components potentially throw two types of exceptions: IOException in case of an I/O failure such as socket timeout or an socket reset and HttpException that signals an HTTP failure such as a violation of the HTTP protocol. Usually I/O errors are considered non-fatal and recoverable, whereas HTTP protocol errors are considered fatal and cannot be automatically recovered from.

所有 HttpCore 组件都可能抛出两种类型的异常：IOException（在 I/O 失败的情况下，如套接字超时或套接字重置）和 HttpException（表示 HTTP 失败，如违反 HTTP 协议）。通常，I/O 错误被认为是非致命的和可恢复的，而 HTTP 协议错误被认为是致命的，不能被自动恢复。

### 2.2.1. Protocol exception

ProtocolException signals a fatal HTTP protocol violation that usually results in an immediate termination of the HTTP message processing.

ProtocolException 标志着一个致命的 HTTP 协议冲突，通常会导致 HTTP 消息处理的立即终止。

## 2.3. Blocking HTTP protocol handlers

### 2.3.1. HTTP service

HttpService is a server side HTTP protocol handler based on the blocking I/O model that implements the essential requirements of the HTTP protocol for the server side message processing as described by RFC 2616.

HttpService 是一个基于阻塞 I/O 模型的服务器端 HTTP 协议处理程序，它实现了 RFC 2616 所描述的服务器端消息处理的 HTTP 协议的基本要求。

HttpService relies on HttpProcessor instance to generate mandatory protocol headers for all outgoing messages and apply common, cross-cutting message transformations to all incoming and outgoing messages, whereas HTTP request handlers are expected to take care of application specific content generation and processing.

HttpService 依赖于 HttpProcessor 实例来为所有传出消息生成强制的协议头，并将通用的、横切的消息转换应用于所有传入和传出消息，而 HTTP 请求处理程序被期望负责特定于应用程序的内容生成和处理。

```
HttpProcessor httpproc = HttpProcessorBuilder.create()
        .add(new ResponseDate())
        .add(new ResponseServer("MyServer-HTTP/1.1"))
        .add(new ResponseContent())
        .add(new ResponseConnControl())
        .build();
HttpService httpService = new HttpService(httpproc, null);
```

#### 2.3.1.1. HTTP request handlers

The HttpRequestHandler interface represents a routine for processing of a specific group of HTTP requests. HttpService is designed to take care of protocol specific aspects, whereas individual request handlers are expected to take care of application specific HTTP processing. The main purpose of a request handler is to generate a response object with a content entity to be sent back to the client in response to the given request.

HttpRequestHandler 接口表示处理一组特定 HTTP 请求的例程。HttpService 被设计用来处理特定于协议的方面，而单个请求处理程序被期望处理特定于应用程序的 HTTP 处理。请求处理程序的主要用途是生成一个响应对象，其中包含一个内容实体，将作为对给定请求的响应发送回客户端。

```
HttpRequestHandler myRequestHandler = new HttpRequestHandler() {

    public void handle(
            HttpRequest request,
            HttpResponse response,
            HttpContext context) throws HttpException, IOException {
        response.setStatusCode(HttpStatus.SC_OK);
        response.setEntity(
                new StringEntity("some important message",
                        ContentType.TEXT_PLAIN));
    }

};
```

#### 2.3.1.2. Request handler resolver

HTTP request handlers are usually managed by a HttpRequestHandlerResolver that matches a request URI to a request handler. HttpCore includes a very simple implementation of the request handler resolver based on a trivial pattern matching algorithm: HttpRequestHandlerRegistry supports only three formats: `*`, `<uri>*` and `*<uri>`

HTTP 请求处理程序通常由与请求处理程序匹配的请求 URI 的 HttpRequestHandlerResolver 管理。HttpCore 包含一个非常简单的请求处理解析器实现，它基于一个简单的模式匹配算法：HttpRequestHandlerRegistry 只支持三种格式：`*`、`<uri>*` 和 `*<uri>`

```
HttpProcessor httpproc = <...>

HttpRequestHandler myRequestHandler1 = <...>
HttpRequestHandler myRequestHandler2 = <...>
HttpRequestHandler myRequestHandler3 = <...>

UriHttpRequestHandlerMapper handlerMapper = new UriHttpRequestHandlerMapper();
handlerMapper.register("/service/*", myRequestHandler1);
handlerMapper.register("*.do", myRequestHandler2);
handlerMapper.register("*", myRequestHandler3);
HttpService httpService = new HttpService(httpproc, handlerMapper);
```

Users are encouraged to provide more sophisticated implementations of HttpRequestHandlerResolver - for instance, based on regular expressions.

我们鼓励用户提供更复杂的 HttpRequestHandlerResolver 实现（例如，基于正则表达式）。

#### 2.3.1.3. Using HTTP service to handle requests

When fully initialized and configured, the HttpService can be used to execute and handle requests for active HTTP connections. The HttpService#handleRequest() method reads an incoming request, generates a response and sends it back to the client. This method can be executed in a loop to handle multiple requests on a persistent connection. The HttpService#handleRequest() method is safe to execute from multiple threads. This allows processing of requests on several connections simultaneously, as long as all the protocol interceptors and requests handlers used by the HttpService are thread-safe.

完全初始化和配置后，HttpService 可用于执行和处理活动 HTTP 连接的请求。HttpService#handleRequest() 方法读取传入请求，生成响应并将其发送回客户机。此方法可以在循环中执行，以处理持久连接上的多个请求。HttpService#handleRequest() 方法可以安全地在多个线程中执行。这允许同时处理多个连接上的请求，只要 HttpService 使用的所有协议拦截器和请求处理程序都是线程安全的。

```
HttpService httpService = <...>
HttpServerConnection conn = <...>
HttpContext context = <...>

boolean active = true;
try {
    while (active && conn.isOpen()) {
        httpService.handleRequest(conn, context);
    }
} finally {
    conn.shutdown();
}
```

### 2.3.2. HTTP request executor

HttpRequestExecutor is a client side HTTP protocol handler based on the blocking I/O model that implements the essential requirements of the HTTP protocol for the client side message processing, as described by RFC 2616. The HttpRequestExecutor relies on the HttpProcessor instance to generate mandatory protocol headers for all outgoing messages and apply common, cross-cutting message transformations to all incoming and outgoing messages. Application specific processing can be implemented outside HttpRequestExecutor once the request has been executed and a response has been received.

HttpRequestExecutor 是一个基于阻塞 I/O 模型的客户端 HTTP 协议处理程序，它实现了 RFC 2616 所描述的客户端消息处理的 HTTP 协议的基本要求。HttpRequestExecutor 依赖于 HttpProcessor 实例来为所有传出消息生成强制的协议头，并将通用的横切消息转换应用于所有传入和传出消息。一旦执行了请求并收到了响应，就可以在 HttpRequestExecutor 之外实现特定于应用程序的处理。

```
HttpClientConnection conn = <...>

HttpProcessor httpproc = HttpProcessorBuilder.create()
        .add(new RequestContent())
        .add(new RequestTargetHost())
        .add(new RequestConnControl())
        .add(new RequestUserAgent("MyClient/1.1"))
        .add(new RequestExpectContinue(true))
        .build();
HttpRequestExecutor httpexecutor = new HttpRequestExecutor();

HttpRequest request = new BasicHttpRequest("GET", "/");
HttpCoreContext context = HttpCoreContext.create();
httpexecutor.preProcess(request, httpproc, context);
HttpResponse response = httpexecutor.execute(request, conn, context);
httpexecutor.postProcess(response, httpproc, context);

HttpEntity entity = response.getEntity();
EntityUtils.consume(entity);
```

Methods of HttpRequestExecutor are safe to execute from multiple threads. This allows execution of requests on several connections simultaneously, as long as all the protocol interceptors used by the HttpRequestExecutor are thread-safe.

HttpRequestExecutor 的方法在多线程中执行是安全的。只要 HttpRequestExecutor 使用的所有协议拦截器都是线程安全的，就允许在几个连接上同时执行请求。

### 2.3.3. Connection persistence / re-use

The ConnectionReuseStrategy interface is intended to determine whether the underlying connection can be re-used for processing of further messages after the transmission of the current message has been completed. The default connection re-use strategy attempts to keep connections alive whenever possible. Firstly, it examines the version of the HTTP protocol used to transmit the message. HTTP/1.1 connections are persistent by default, while HTTP/1.0 connections are not. Secondly, it examines the value of the Connection header. The peer can indicate whether it intends to re-use the connection on the opposite side by sending Keep-Alive or Close values in the Connection header. Thirdly, the strategy makes the decision whether the connection is safe to re-use based on the properties of the enclosed entity, if available.

ConnectionReuseStrategy 接口用于确定在完成当前消息的传输之后，是否可以重用底层连接来处理进一步的消息。默认的连接重用策略尝试尽可能保持连接处于活动状态。首先，它检查用于传输消息的 HTTP 协议的版本。HTTP/1.1 连接在默认情况下是持久的，而 HTTP/1.0 连接则不是。其次，它检查连接头的值。对等方可以通过在连接头中发送 Keep-Alive 或 Close 值来指示它是否打算重用另一端的连接。第三，该策略根据封闭实体的属性（如果可用）来决定连接是否可以安全重用。

## 2.4. Connection pools

Efficient client-side HTTP transports often requires effective re-use of persistent connections. HttpCore facilitates the process of connection re-use by providing support for managing pools of persistent HTTP connections. Connection pool implementations are thread-safe and can be used concurrently by multiple consumers.

高效的客户端 HTTP 传输通常需要有效地重用持久连接。通过提供对管理持久 HTTP 连接池的支持，HttpCore 简化了连接重用的过程。连接池实现是线程安全的，可以由多个使用者并发使用。

By default the pool allows only 20 concurrent connections in total and two concurrent connections per a unique route. The two connection limit is due to the requirements of the HTTP specification. However, in practical terms this can often be too restrictive. One can change the pool configuration at runtime to allow for more concurrent connections depending on a particular application context.

默认情况下，池总共只允许 20 个并发连接，每个唯一路由只允许两个并发连接。这两个连接限制是由于 HTTP 规范的要求。然而，在实际应用中，这通常会有太多的限制。可以在运行时更改池配置，以根据特定的应用程序上下文允许更多的并发连接。

```
HttpHost target = new HttpHost("localhost");
BasicConnPool connpool = new BasicConnPool();
connpool.setMaxTotal(200);
connpool.setDefaultMaxPerRoute(10);
connpool.setMaxPerRoute(target, 20);
Future<BasicPoolEntry> future = connpool.lease(target, null);
BasicPoolEntry poolEntry = future.get();
HttpClientConnection conn = poolEntry.getConnection();
```

Please note that the connection pool has no way of knowing whether or not a leased connection is still being used. It is the responsibility of the connection pool user to ensure that the connection is released back to the pool once it is not longer needed, even if the connection is not reusable.

请注意，连接池无法知道已租用的连接是否仍在使用。连接池用户有责任确保在不再需要连接时将其释放回池中，即使该连接不是可重用的。

```
BasicConnPool connpool = <...>
Future<BasicPoolEntry> future = connpool.lease(target, null);
BasicPoolEntry poolEntry = future.get();
try {
    HttpClientConnection conn = poolEntry.getConnection();
} finally {
    connpool.release(poolEntry, true);
}
```

The state of the connection pool can be interrogated at runtime.

可以在运行时查询连接池的状态。

```
HttpHost target = new HttpHost("localhost");
BasicConnPool connpool = <...>
PoolStats totalStats = connpool.getTotalStats();
System.out.println("total available: " + totalStats.getAvailable());
System.out.println("total leased: " + totalStats.getLeased());
System.out.println("total pending: " + totalStats.getPending());
PoolStats targetStats = connpool.getStats(target);
System.out.println("target available: " + targetStats.getAvailable());
System.out.println("target leased: " + targetStats.getLeased());
System.out.println("target pending: " + targetStats.getPending());
```

Please note that connection pools do not pro-actively evict expired connections. Even though expired connection cannot be leased to the requester, the pool may accumulate stale connections over time especially after a period of inactivity. It is generally advisable to force eviction of expired and idle connections from the pool after an extensive period of inactivity.

请注意，连接池不会主动地退出过期的连接。即使不能将过期的连接租给请求者，池也会随着时间积累过期的连接，尤其是在一段时间不活动之后。通常建议在长时间不活动之后强制从池中清除过期和空闲连接。

```
BasicConnPool connpool = <...>
connpool.closeExpired();
connpool.closeIdle(1, TimeUnit.MINUTES);
```

Generally it is considered to be a responsibility of the consumer to keep track of connections leased from the pool and to ensure their immediate release as soon as they are no longer needed or actively used. Nevertheless BasicConnPool provides protected methods to enumerate available idle connections and those currently leased from the pool. This enables the pool consumer to query connection state and selectively terminate connections meeting a particular criterion.

通常认为，使用者有责任跟踪从池中租借的连接，并确保在不再需要或不再积极使用连接时立即释放它们。然而，BasicConnPool 提供了 protected 方法来枚举可用的空闲连接和当前从池中租用的连接。这使池使用者能够查询连接状态并有选择地终止满足特定条件的连接。

```
static class MyBasicConnPool extends BasicConnPool {

    @Override
    protected void enumAvailable(final PoolEntryCallback<HttpHost, HttpClientConnection> callback) {
        super.enumAvailable(callback);
    }

    @Override
    protected void enumLeased(final PoolEntryCallback<HttpHost, HttpClientConnection> callback) {
        super.enumLeased(callback);
    }

}
```

```
MyBasicConnPool connpool = new MyBasicConnPool();
connpool.enumAvailable(new PoolEntryCallback<HttpHost, HttpClientConnection>() {

    @Override
    public void process(final PoolEntry<HttpHost, HttpClientConnection> entry) {
        Date creationTime = new Date(entry.getCreated());
        if (creationTime.before(someTime)) {
            entry.close();
        }
    }

});
```

## 2.5. TLS/SSL support

Blocking connections can be bound to any arbitrary socket. This makes SSL support quite straight-forward. Any SSLSocket instance can be bound to a blocking connection in order to make all messages transmitted over than connection secured by TLS/SSL.

阻塞连接可以绑定到任意的套接字。这使得 SSL 支持非常简单。任何 SSLSocket 实例都可以绑定到一个阻塞连接，以便使所有通过连接传输的消息都不受 TLS/SSL 的保护。

```
SSLContext sslcontext = SSLContexts.createSystemDefault();
SocketFactory sf = sslcontext.getSocketFactory();
SSLSocket socket = (SSLSocket) sf.createSocket("somehost", 443);
// Enforce TLS and disable SSL
socket.setEnabledProtocols(new String[] {
        "TLSv1",
        "TLSv1.1",
        "TLSv1.2" });
// Enforce strong ciphers
socket.setEnabledCipherSuites(new String[] {
        "TLS_RSA_WITH_AES_256_CBC_SHA",
        "TLS_DHE_RSA_WITH_AES_256_CBC_SHA",
        "TLS_DHE_DSS_WITH_AES_256_CBC_SHA" });
DefaultBHttpClientConnection conn = new DefaultBHttpClientConnection(8 * 1204);
conn.bind(socket);
```

## 2.6. Embedded HTTP server

As of version 4.4 HttpCore ships with an embedded HTTP server based on blocking I/O components described above.

从版本 4.4 开始，HttpCore 附带了一个基于上述阻塞 I/O 组件的嵌入式 HTTP 服务器。

```
HttpRequestHandler requestHandler = <...>
HttpProcessor httpProcessor = <...>
SocketConfig socketConfig = SocketConfig.custom()
        .setSoTimeout(15000)
        .setTcpNoDelay(true)
        .build();
final HttpServer server = ServerBootstrap.bootstrap()
        .setListenerPort(8080)
        .setHttpProcessor(httpProcessor)
        .setSocketConfig(socketConfig)
        .setExceptionLogger(new StdErrorExceptionLogger())
        .registerHandler("*", requestHandler)
        .create();
server.start();
server.awaitTermination(Long.MAX_VALUE, TimeUnit.DAYS);

Runtime.getRuntime().addShutdownHook(new Thread() {
    @Override
    public void run() {
        server.shutdown(5, TimeUnit.SECONDS);
    }
});
```
