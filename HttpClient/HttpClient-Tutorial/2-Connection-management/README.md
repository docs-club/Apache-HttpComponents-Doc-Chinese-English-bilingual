# Chapter 2 Connection management（连接管理）

## 2.1 Connection persistence（连接的持久性）
The process of establishing a connection from one host to another is quite complex and involves multiple packet exchanges between two endpoints, which can be quite time consuming. The overhead of connection handshaking can be significant, especially for small HTTP messages. One can achieve a much higher data throughput if open connections can be re-used to execute multiple requests.

HTTP/1.1 states that HTTP connections can be re-used for multiple requests per default. HTTP/1.0 compliant endpoints can also use a mechanism to explicitly communicate their preference to keep connection alive and use it for multiple requests. HTTP agents can also keep idle connections alive for a certain period time in case a connection to the same target host is needed for subsequent requests. The ability to keep connections alive is usually refered to as connection persistence. HttpClient fully supports connection persistence.

## 2.2 HTTP connection routing
HttpClient is capable of establishing connections to the target host either directly or via a route that may involve multiple intermediate connections - also referred to as hops. HttpClient differentiates connections of a route into plain, tunneled and layered. The use of multiple intermediate proxies to tunnel connections to the target host is referred to as proxy chaining.

Plain routes are established by connecting to the target or the first and only proxy. Tunnelled routes are established by connecting to the first and tunnelling through a chain of proxies to the target. Routes without a proxy cannot be tunnelled. Layered routes are established by layering a protocol over an existing connection. Protocols can only be layered over a tunnel to the target, or over a direct connection without proxies.

### 2.2.1 Route computation
The `RouteInfo` interface represents information about a definitive route to a target host involving one or more intermediate steps or hops. `HttpRoute` is a concrete implementation of the `RouteInfo`, which cannot be changed (is immutable). `HttpTracker` is a mutable `RouteInfo` implementation used internally by HttpClient to track the remaining hops to the ultimate route target. `HttpTracker` can be updated after a successful execution of the next hop towards the route target. `HttpRouteDirector` is a helper class that can be used to compute the next step in a route. This class is used internally by HttpClient.

`HttpRoutePlanner` is an interface representing a strategy to compute a complete route to a given target based on the execution context. HttpClient ships with two default `HttpRoutePlanner` implementations. `SystemDefaultRoutePlanner` is based on `java.net.ProxySelector`. By default, it will pick up the proxy settings of the JVM, either from system properties or from the browser running the application. The `DefaultProxyRoutePlanner` implementation does not make use of any Java system properties, nor any system or browser proxy settings. It always computes routes via the same default proxy.

### 2.2.2 Secure HTTP connections
HTTP connections can be considered secure if information transmitted between two connection endpoints cannot be read or tampered with by an unauthorized third party. The SSL/TLS protocol is the most widely used technique to ensure HTTP transport security. However, other encryption techniques could be employed as well. Usually, HTTP transport is layered over the SSL/TLS encrypted connection.

## 2.3 HTTP connection managers
### 2.3.1 Managed connections and connection managers
HTTP connections are complex, stateful, thread-unsafe objects which need to be properly managed to function correctly. HTTP connections can only be used by one execution thread at a time. HttpClient employs a special entity to manage access to HTTP connections called HTTP connection manager and represented by the `HttpClientConnectionManager` interface. The purpose of an HTTP connection manager is to serve as a factory for new HTTP connections, to manage life cycle of persistent connections and to synchronize access to persistent connections making sure that only one thread can have access to a connection at a time. Internally HTTP connection managers work with instances of `ManagedHttpClientConnection` acting as a proxy for a real connection that manages connection state and controls execution of I/O operations. If a managed connection is released or get explicitly closed by its consumer the underlying connection gets detached from its proxy and is returned back to the manager. Even though the service consumer still holds a reference to the proxy instance, it is no longer able to execute any I/O operations or change the state of the real connection either intentionally or unintentionally.

This is an example of acquiring a connection from a connection manager:

```
HttpClientContext context = HttpClientContext.create();
HttpClientConnectionManager connMrg = new BasicHttpClientConnectionManager();
HttpRoute route = new HttpRoute(new HttpHost("localhost", 80));
// Request new connection. This can be a long process
ConnectionRequest connRequest = connMrg.requestConnection(route, null);
// Wait for connection up to 10 sec
HttpClientConnection conn = connRequest.get(10, TimeUnit.SECONDS);
try {
    // If not open
    if (!conn.isOpen()) {
        // establish connection based on its route info
        connMrg.connect(conn, route, 1000, context);
        // and mark it as route complete
        connMrg.routeComplete(conn, route, context);
    }
    // Do useful things with the connection.
} finally {
    connMrg.releaseConnection(conn, null, 1, TimeUnit.MINUTES);
}
```

The connection request can be terminated prematurely by calling `ConnectionRequest#cancel()` if necessary. This will unblock the thread blocked in the `ConnectionRequest#get()` method.

### 2.3.2 Simple connection manager
`BasicHttpClientConnectionManager` is a simple connection manager that maintains only one connection at a time. Even though this class is thread-safe it ought to be used by one execution thread only. `BasicHttpClientConnectionManager` will make an effort to reuse the connection for subsequent requests with the same route. It will, however, close the existing connection and re-open it for the given route, if the route of the persistent connection does not match that of the connection request. If the connection has been already been allocated, then `java.lang.IllegalStateException` is thrown.

This connection manager implementation should be used inside an EJB container.

### 2.3.3 Pooling connection manager
`PoolingHttpClientConnectionManager` is a more complex implementation that manages a pool of client connections and is able to service connection requests from multiple execution threads. Connections are pooled on a per route basis. A request for a route for which the manager already has a persistent connection available in the pool will be serviced by leasing a connection from the pool rather than creating a brand new connection.

`PoolingHttpClientConnectionManager` maintains a maximum limit of connections on a per route basis and in total. Per default this implementation will create no more than 2 concurrent connections per given route and no more 20 connections in total. For many real-world applications these limits may prove too constraining, especially if they use HTTP as a transport protocol for their services.

This example shows how the connection pool parameters can be adjusted:

```
PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
// Increase max total connection to 200
cm.setMaxTotal(200);
// Increase default max connection per route to 20
cm.setDefaultMaxPerRoute(20);
// Increase max connections for localhost:80 to 50
HttpHost localhost = new HttpHost("locahost", 80);
cm.setMaxPerRoute(new HttpRoute(localhost), 50);

CloseableHttpClient httpClient = HttpClients.custom()
        .setConnectionManager(cm)
        .build();
```

### 2.3.4 Connection manager shutdown
When an HttpClient instance is no longer needed and is about to go out of scope it is important to shut down its connection manager to ensure that all connections kept alive by the manager get closed and system resources allocated by those connections are released.

```
CloseableHttpClient httpClient = <...>
httpClient.close();
```

## 2.4 Multithreaded request execution
When equipped with a pooling connection manager such as `PoolingClientConnectionManager`, HttpClient can be used to execute multiple requests simultaneously using multiple threads of execution.

The `PoolingClientConnectionManager` will allocate connections based on its configuration. If all connections for a given route have already been leased, a request for a connection will block until a connection is released back to the pool. One can ensure the connection manager does not block indefinitely in the connection request operation by setting `'http.conn-manager.timeout'` to a positive value. If the connection request cannot be serviced within the given time period `ConnectionPoolTimeoutException` will be thrown.

```
PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
CloseableHttpClient httpClient = HttpClients.custom()
        .setConnectionManager(cm)
        .build();

// URIs to perform GETs on
String[] urisToGet = {
    "http://www.domain1.com/",
    "http://www.domain2.com/",
    "http://www.domain3.com/",
    "http://www.domain4.com/"
};

// create a thread for each URI
GetThread[] threads = new GetThread[urisToGet.length];
for (int i = 0; i < threads.length; i++) {
    HttpGet httpget = new HttpGet(urisToGet[i]);
    threads[i] = new GetThread(httpClient, httpget);
}

// start the threads
for (int j = 0; j < threads.length; j++) {
    threads[j].start();
}

// join the threads
for (int j = 0; j < threads.length; j++) {
    threads[j].join();
}
```

While `HttpClient` instances are thread safe and can be shared between multiple threads of execution, it is highly recommended that each thread maintains its own dedicated instance of `HttpContext`.

```
static class GetThread extends Thread {

    private final CloseableHttpClient httpClient;
    private final HttpContext context;
    private final HttpGet httpget;

    public GetThread(CloseableHttpClient httpClient, HttpGet httpget) {
        this.httpClient = httpClient;
        this.context = HttpClientContext.create();
        this.httpget = httpget;
    }

    @Override
    public void run() {
        try {
            CloseableHttpResponse response = httpClient.execute(
                    httpget, context);
            try {
                HttpEntity entity = response.getEntity();
            } finally {
                response.close();
            }
        } catch (ClientProtocolException ex) {
            // Handle protocol errors
        } catch (IOException ex) {
            // Handle I/O errors
        }
    }

}
```

### 2.5 Connection eviction policy
One of the major shortcomings of the classic blocking I/O model is that the network socket can react to I/O events only when blocked in an I/O operation. When a connection is released back to the manager, it can be kept alive however it is unable to monitor the status of the socket and react to any I/O events. If the connection gets closed on the server side, the client side connection is unable to detect the change in the connection state (and react appropriately by closing the socket on its end).

HttpClient tries to mitigate the problem by testing whether the connection is 'stale', that is no longer valid because it was closed on the server side, prior to using the connection for executing an HTTP request. The stale connection check is not 100% reliable. The only feasible solution that does not involve a one thread per socket model for idle connections is a dedicated monitor thread used to evict connections that are considered expired due to a long period of inactivity. The monitor thread can periodically call `ClientConnectionManager#closeExpiredConnections()` method to close all expired connections and evict closed connections from the pool. It can also optionally call `ClientConnectionManager#closeIdleConnections()` method to close all connections that have been idle over a given period of time.

```
public static class IdleConnectionMonitorThread extends Thread {

    private final HttpClientConnectionManager connMgr;
    private volatile boolean shutdown;

    public IdleConnectionMonitorThread(HttpClientConnectionManager connMgr) {
        super();
        this.connMgr = connMgr;
    }

    @Override
    public void run() {
        try {
            while (!shutdown) {
                synchronized (this) {
                    wait(5000);
                    // Close expired connections
                    connMgr.closeExpiredConnections();
                    // Optionally, close connections
                    // that have been idle longer than 30 sec
                    connMgr.closeIdleConnections(30, TimeUnit.SECONDS);
                }
            }
        } catch (InterruptedException ex) {
            // terminate
        }
    }

    public void shutdown() {
        shutdown = true;
        synchronized (this) {
            notifyAll();
        }
    }

}
```

## 2.6 Connection keep alive strategy
The HTTP specification does not specify how long a persistent connection may be and should be kept alive. Some HTTP servers use a non-standard `Keep-Alive` header to communicate to the client the period of time in seconds they intend to keep the connection alive on the server side. HttpClient makes use of this information if available. If the `Keep-Alive` header is not present in the response, HttpClient assumes the connection can be kept alive indefinitely. However, many HTTP servers in general use are configured to drop persistent connections after a certain period of inactivity in order to conserve system resources, quite often without informing the client. In case the default strategy turns out to be too optimistic, one may want to provide a custom keep-alive strategy.

```
ConnectionKeepAliveStrategy myStrategy = new ConnectionKeepAliveStrategy() {

    public long getKeepAliveDuration(HttpResponse response, HttpContext context) {
        // Honor 'keep-alive' header
        HeaderElementIterator it = new BasicHeaderElementIterator(
                response.headerIterator(HTTP.CONN_KEEP_ALIVE));
        while (it.hasNext()) {
            HeaderElement he = it.nextElement();
            String param = he.getName();
            String value = he.getValue();
            if (value != null && param.equalsIgnoreCase("timeout")) {
                try {
                    return Long.parseLong(value) * 1000;
                } catch(NumberFormatException ignore) {
                }
            }
        }
        HttpHost target = (HttpHost) context.getAttribute(
                HttpClientContext.HTTP_TARGET_HOST);
        if ("www.naughty-server.com".equalsIgnoreCase(target.getHostName())) {
            // Keep alive for 5 seconds only
            return 5 * 1000;
        } else {
            // otherwise keep alive for 30 seconds
            return 30 * 1000;
        }
    }

};
CloseableHttpClient client = HttpClients.custom()
        .setKeepAliveStrategy(myStrategy)
        .build();
```

## 2.7 Connection socket factories
HTTP connections make use of a `java.net.Socket` object internally to handle transmission of data across the wire. However they rely on the `ConnectionSocketFactory` interface to create, initialize and connect sockets. This enables the users of HttpClient to provide application specific socket initialization code at runtime. `PlainConnectionSocketFactory` is the default factory for creating and initializing plain (unencrypted) sockets.

The process of creating a socket and that of connecting it to a host are decoupled, so that the socket could be closed while being blocked in the connect operation.

```
HttpClientContext clientContext = HttpClientContext.create();
PlainConnectionSocketFactory sf = PlainConnectionSocketFactory.getSocketFactory();
Socket socket = sf.createSocket(clientContext);
int timeout = 1000; //ms
HttpHost target = new HttpHost("localhost");
InetSocketAddress remoteAddress = new InetSocketAddress(
        InetAddress.getByAddress(new byte[] {127,0,0,1}), 80);
sf.connectSocket(timeout, socket, target, remoteAddress, null, clientContext);
```

### 2.7.1 Secure socket layering
`LayeredConnectionSocketFactory` is an extension of the `ConnectionSocketFactory` interface. Layered socket factories are capable of creating sockets layered over an existing plain socket. Socket layering is used primarily for creating secure sockets through proxies. HttpClient ships with `SSLSocketFactory` that implements SSL/TLS layering. Please note HttpClient does not use any custom encryption functionality. It is fully reliant on standard Java Cryptography (JCE) and Secure Sockets (JSEE) extensions.

### 2.7.2 Integration with connection manager
Custom connection socket factories can be associated with a particular protocol scheme as as HTTP or HTTPS and then used to create a custom connection manager.

```
ConnectionSocketFactory plainsf = <...>
LayeredConnectionSocketFactory sslsf = <...>
Registry<ConnectionSocketFactory> r = RegistryBuilder.<ConnectionSocketFactory>create()
        .register("http", plainsf)
        .register("https", sslsf)
        .build();

HttpClientConnectionManager cm = new PoolingHttpClientConnectionManager(r);
HttpClients.custom()
        .setConnectionManager(cm)
        .build();
```

### 2.7.3 SSL/TLS customization
HttpClient makes use of `SSLConnectionSocketFactory` to create SSL connections. `SSLConnectionSocketFactory` allows for a high degree of customization. It can take an instance of `javax.net.ssl.SSLContext` as a parameter and use it to create custom configured SSL connections.

```
KeyStore myTrustStore = <...>
SSLContext sslContext = SSLContexts.custom()
        .loadTrustMaterial(myTrustStore)
        .build();
SSLConnectionSocketFactory sslsf = new SSLConnectionSocketFactory(sslContext);
```

Customization of `SSLConnectionSocketFactory` implies a certain degree of familiarity with the concepts of the SSL/TLS protocol, a detailed explanation of which is out of scope for this document. Please refer to the `Java™ Secure Socket Extension (JSSE) Reference Guide` for a detailed description of `javax.net.ssl.SSLContext` and related tools.

### 2.7.4 Hostname verification
In addition to the trust verification and the client authentication performed on the SSL/TLS protocol level, HttpClient can optionally verify whether the target hostname matches the names stored inside the server's X.509 certificate, once the connection has been established. This verification can provide additional guarantees of authenticity of the server trust material. The `javax.net.ssl.HostnameVerifier` interface represents a strategy for hostname verification. HttpClient ships with two `javax.net.ssl.HostnameVerifier` implementations. Important: hostname verification should not be confused with SSL trust verification.

- DefaultHostnameVerifier:  The default implementation used by HttpClient is expected to be compliant with RFC 2818. The hostname must match any of alternative names specified by the certificate, or in case no alternative names are given the most specific CN of the certificate subject. A wildcard can occur in the CN, and in any of the subject-alts.

- NoopHostnameVerifier:  This hostname verifier essentially turns hostname verification off. It accepts any SSL session as valid and matching the target host.

Per default HttpClient uses the `DefaultHostnameVerifier` implementation. One can specify a different hostname verifier implementation if desired

```
SSLContext sslContext = SSLContexts.createSystemDefault();
SSLConnectionSocketFactory sslsf = new SSLConnectionSocketFactory(sslContext, NoopHostnameVerifier.INSTANCE);
```

As of version 4.4 HttpClient uses the public suffix list kindly maintained by Mozilla Foundation to make sure that wildcards in SSL certificates cannot be misused to apply to multiple domains with a common top-level domain. HttpClient ships with a copy of the list retrieved at the time of the release. The latest revision of the list can found at https://publicsuffix.org/list/. It is highly adviseable to make a local copy of the list and download the list no more than once per day from its original location.

```
PublicSuffixMatcher publicSuffixMatcher = PublicSuffixMatcherLoader
        .load(PublicSuffixMatcher.class.getResource("my-copy-effective_tld_names.dat"));
DefaultHostnameVerifier hostnameVerifier = new DefaultHostnameVerifier(publicSuffixMatcher);
```

One can disable verification against the public suffic list by using null matcher.

```
DefaultHostnameVerifier hostnameVerifier = new DefaultHostnameVerifier(null);
```

## 2.8 HttpClient proxy configuration
Even though HttpClient is aware of complex routing schemes and proxy chaining, it supports only simple direct or one hop proxy connections out of the box.

The simplest way to tell HttpClient to connect to the target host via a proxy is by setting the default proxy parameter:

```
HttpHost proxy = new HttpHost("someproxy", 8080);
DefaultProxyRoutePlanner routePlanner = new DefaultProxyRoutePlanner(proxy);
CloseableHttpClient httpclient = HttpClients.custom()
        .setRoutePlanner(routePlanner)
        .build();
```

One can also instruct HttpClient to use the standard JRE proxy selector to obtain proxy information:

```
SystemDefaultRoutePlanner routePlanner = new SystemDefaultRoutePlanner(ProxySelector.getDefault());
CloseableHttpClient httpclient = HttpClients.custom()
        .setRoutePlanner(routePlanner)
        .build();
```

Alternatively, one can provide a custom RoutePlanner implementation in order to have a complete control over the process of HTTP route computation:

```
HttpRoutePlanner routePlanner = new HttpRoutePlanner() {

    public HttpRoute determineRoute(
            HttpHost target,
            HttpRequest request,
            HttpContext context) throws HttpException {
        return new HttpRoute(target, null,  new HttpHost("someproxy", 8080),
                "https".equalsIgnoreCase(target.getSchemeName()));
    }

};
CloseableHttpClient httpclient = HttpClients.custom()
        .setRoutePlanner(routePlanner)
        .build();
    }
}
```
