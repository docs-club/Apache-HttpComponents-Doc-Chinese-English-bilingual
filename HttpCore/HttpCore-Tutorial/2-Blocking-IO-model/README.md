# Chapter 2. Blocking I/O model

Blocking (or classic) I/O in Java represents a highly efficient and convenient I/O model well suited for high performance applications where the number of concurrent connections is relatively moderate. Modern JVMs are capable of efficient context switching and the blocking I/O model should offer the best performance in terms of raw data throughput as long as the number of concurrent connections is below one thousand and connections are mostly busy transmitting data. However for applications where connections stay idle most of the time the overhead of context switching may become substantial and a non-blocking I/O model may present a better alternative.

## 2.1. Blocking HTTP connections

HTTP connections are responsible for HTTP message serialization and deserialization. One should rarely need to use HTTP connection objects directly. There are higher level protocol components intended for execution and processing of HTTP requests. However, in some cases direct interaction with HTTP connections may be necessary, for instance, to access properties such as the connection status, the socket timeout or the local and remote addresses.

It is important to bear in mind that HTTP connections are not thread-safe. We strongly recommend limiting all interactions with HTTP connection objects to one thread. The only method of HttpConnection interface and its sub-interfaces which is safe to invoke from another thread is HttpConnection#shutdown() .

### 2.1.1. Working with blocking HTTP connections

HttpCore does not provide full support for opening connections because the process of establishing a new connection - especially on the client side - can be very complex when it involves one or more authenticating or/and tunneling proxies. Instead, blocking HTTP connections can be bound to any arbitrary network socket.

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

Over-simplified process of request execution on the client side may look like this:

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

### 2.1.2. Content transfer with blocking I/O

HTTP connections manage the process of the content transfer using the HttpEntity interface. HTTP connections generate an entity object that encapsulates the content stream of the incoming message. Please note that HttpServerConnection#receiveRequestEntity() and HttpClientConnection#receiveResponseEntity() do not retrieve or buffer any incoming data. They merely inject an appropriate content codec based on the properties of the incoming message. The content can be retrieved by reading from the content input stream of the enclosed entity using HttpEntity#getContent(). The incoming data will be decoded automatically and completely transparently to the data consumer. Likewise, HTTP connections rely on HttpEntity#writeTo(OutputStream) method to generate the content of an outgoing message. If an outgoing message encloses an entity, the content will be encoded automatically based on the properties of the message.

### 2.1.3. Supported content transfer mechanisms

Default implementations of HTTP connections support three content transfer mechanisms defined by the HTTP/1.1 specification:

- Content-Length delimited: The end of the content entity is determined by the value of the Content-Length header. Maximum entity length: Long#MAX_VALUE.

- Identity coding: The end of the content entity is demarcated by closing the underlying connection (end of stream condition). For obvious reasons the identity encoding can only be used on the server side. Maximum entity length: unlimited.

- Chunk coding: The content is sent in small chunks. Maximum entity length: unlimited.

The appropriate content stream class will be created automatically depending on properties of the entity enclosed with the message.

### 2.1.4. Terminating HTTP connections

HTTP connections can be terminated either gracefully by calling HttpConnection#close() or forcibly by calling HttpConnection#shutdown(). The former tries to flush all buffered data prior to terminating the connection and may block indefinitely. The HttpConnection#close() method is not thread-safe. The latter terminates the connection without flushing internal buffers and returns control to the caller as soon as possible without blocking for long. The HttpConnection#shutdown() method is thread-safe.

## 2.2. HTTP exception handling

All HttpCore components potentially throw two types of exceptions: IOException in case of an I/O failure such as socket timeout or an socket reset and HttpException that signals an HTTP failure such as a violation of the HTTP protocol. Usually I/O errors are considered non-fatal and recoverable, whereas HTTP protocol errors are considered fatal and cannot be automatically recovered from.

### 2.2.1. Protocol exception

ProtocolException signals a fatal HTTP protocol violation that usually results in an immediate termination of the HTTP message processing.

## 2.3. Blocking HTTP protocol handlers

### 2.3.1. HTTP service

HttpService is a server side HTTP protocol handler based on the blocking I/O model that implements the essential requirements of the HTTP protocol for the server side message processing as described by RFC 2616.

HttpService relies on HttpProcessor instance to generate mandatory protocol headers for all outgoing messages and apply common, cross-cutting message transformations to all incoming and outgoing messages, whereas HTTP request handlers are expected to take care of application specific content generation and processing.

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

HTTP request handlers are usually managed by a HttpRequestHandlerResolver that matches a request URI to a request handler. HttpCore includes a very simple implementation of the request handler resolver based on a trivial pattern matching algorithm: HttpRequestHandlerRegistry supports only three formats: _, <uri>_ and \*<uri>.

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

#### 2.3.1.3. Using HTTP service to handle requests

When fully initialized and configured, the HttpService can be used to execute and handle requests for active HTTP connections. The HttpService#handleRequest() method reads an incoming request, generates a response and sends it back to the client. This method can be executed in a loop to handle multiple requests on a persistent connection. The HttpService#handleRequest() method is safe to execute from multiple threads. This allows processing of requests on several connections simultaneously, as long as all the protocol interceptors and requests handlers used by the HttpService are thread-safe.

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

### 2.3.3. Connection persistence / re-use

The ConnectionReuseStrategy interface is intended to determine whether the underlying connection can be re-used for processing of further messages after the transmission of the current message has been completed. The default connection re-use strategy attempts to keep connections alive whenever possible. Firstly, it examines the version of the HTTP protocol used to transmit the message. HTTP/1.1 connections are persistent by default, while HTTP/1.0 connections are not. Secondly, it examines the value of the Connection header. The peer can indicate whether it intends to re-use the connection on the opposite side by sending Keep-Alive or Close values in the Connection header. Thirdly, the strategy makes the decision whether the connection is safe to re-use based on the properties of the enclosed entity, if available.

## 2.4. Connection pools

Efficient client-side HTTP transports often requires effective re-use of persistent connections. HttpCore facilitates the process of connection re-use by providing support for managing pools of persistent HTTP connections. Connection pool implementations are thread-safe and can be used concurrently by multiple consumers.

By default the pool allows only 20 concurrent connections in total and two concurrent connections per a unique route. The two connection limit is due to the requirements of the HTTP specification. However, in practical terms this can often be too restrictive. One can change the pool configuration at runtime to allow for more concurrent connections depending on a particular application context.

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

```
BasicConnPool connpool = <...>
connpool.closeExpired();
connpool.closeIdle(1, TimeUnit.MINUTES);
```

Generally it is considered to be a responsibility of the consumer to keep track of connections leased from the pool and to ensure their immediate release as soon as they are no longer needed or actively used. Nevertheless BasicConnPool provides protected methods to enumerate available idle connections and those currently leased from the pool. This enables the pool consumer to query connection state and selectively terminate connections meeting a particular criterion.

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
