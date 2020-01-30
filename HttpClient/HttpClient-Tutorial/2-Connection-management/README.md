# Chapter 2. Connection management

## 2.1 Connection persistence

The process of establishing a connection from one host to another is quite complex and involves multiple packet exchanges between two endpoints, which can be quite time consuming. The overhead of connection handshaking can be significant, especially for small HTTP messages. One can achieve a much higher data throughput if open connections can be re-used to execute multiple requests.

从一台主机到另一台主机建立连接的过程相当复杂，涉及两个端点之间的多个包交换，这可能非常耗时。连接握手的开销非常大，尤其是对于小的 HTTP 消息而言。如果可以重用打开的连接来执行多个请求，则可以获得更高的数据吞吐量。

HTTP/1.1 states that HTTP connections can be re-used for multiple requests per default. HTTP/1.0 compliant endpoints can also use a mechanism to explicitly communicate their preference to keep connection alive and use it for multiple requests. HTTP agents can also keep idle connections alive for a certain period time in case a connection to the same target host is needed for subsequent requests. The ability to keep connections alive is usually refered to as connection persistence. HttpClient fully supports connection persistence.

HTTP/1.1 声明的 HTTP 连接默认情况下可以被多个请求重用。符合 HTTP/1.0 的端点还可以使用一种机制显式地通信它们的首选项，以保持连接活动，并将其用于多个请求。HTTP 代理还可以将空闲连接保持一定时间的活动状态，以防后续请求需要连接到相同的目标主机。保持连接活动的能力通常称为连接持久性。HttpClient 完全支持连接持久性。

## 2.2 HTTP connection routing

HttpClient is capable of establishing connections to the target host either directly or via a route that may involve multiple intermediate connections - also referred to as hops. HttpClient differentiates connections of a route into plain, tunneled and layered. The use of multiple intermediate proxies to tunnel connections to the target host is referred to as proxy chaining.

HttpClient 能够直接建立到目标主机的连接，或通过路由来达到目的，但这种方式可能涉及多个中间连接（也称为 hop）。HttpClient 将路由的连接分为普通连接、隧道连接和分层连接。使用多个中间代理来进行到目标主机的隧道连接称为代理链接。

Plain routes are established by connecting to the target or the first and only proxy. Tunnelled routes are established by connecting to the first and tunnelling through a chain of proxies to the target. Routes without a proxy cannot be tunnelled. Layered routes are established by layering a protocol over an existing connection. Protocols can only be layered over a tunnel to the target, or over a direct connection without proxies.

普通路由是通过连接到目标或代理（它是第一个也是唯一一个）来建立的。隧道路由是通过连接到第一个代理，并通过代理链隧道连接至目标来建立的。没有代理的路由不能被隧道化。分层路由是通过在现有连接上分层协议来建立的。协议只能在连接至目标的隧道上分层，或者在没有代理的直接连接上分层。

### 2.2.1 Route computation

The RouteInfo interface represents information about a definitive route to a target host involving one or more intermediate steps or hops. HttpRoute is a concrete implementation of the RouteInfo, which cannot be changed (is immutable). HttpTracker is a mutable RouteInfo implementation used internally by HttpClient to track the remaining hops to the ultimate route target. HttpTracker can be updated after a successful execution of the next hop towards the route target. HttpRouteDirector is a helper class that can be used to compute the next step in a route. This class is used internally by HttpClient.

RouteInfo 接口表示关于到目标主机的最终路由的信息，该路由涉及一个或多个中间步骤或跃点。HttpRoute 是 RouteInfo 的一个具体实现，它不能被更改（不可变类）。HttpTracker 是一个可变的 RouteInfo 实现，HttpClient 内部使用它来跟踪剩余的跳转到最终路由目标。HttpTracker 可以在成功执行下一个跳转到路由目标后更新。HttpRouteDirector 是一个助手类，可以用来计算路由的下一步。这个类由 HttpClient 在内部使用。

HttpRoutePlanner is an interface representing a strategy to compute a complete route to a given target based on the execution context. HttpClient ships with two default HttpRoutePlanner implementations. SystemDefaultRoutePlanner is based on java.net.ProxySelector. By default, it will pick up the proxy settings of the JVM, either from system properties or from the browser running the application. The DefaultProxyRoutePlanner implementation does not make use of any Java system properties, nor any system or browser proxy settings. It always computes routes via the same default proxy.

HttpRoutePlanner 是一个接口，它表示根据执行上下文计算到给定目标的完整路由的策略。HttpClient 附带两个默认的 HttpRoutePlanner 实现。SystemDefaultRoutePlanner 基于 java.net.ProxySelector。默认情况下，它将从系统属性或运行应用程序的浏览器获取 JVM 的代理设置。DefaultProxyRoutePlanner 实现不使用任何 Java 系统属性，也不使用任何系统或浏览器代理设置。它总是通过相同的默认代理计算路由。

### 2.2.2 Secure HTTP connections

HTTP connections can be considered secure if information transmitted between two connection endpoints cannot be read or tampered with by an unauthorized third party. The SSL/TLS protocol is the most widely used technique to ensure HTTP transport security. However, other encryption techniques could be employed as well. Usually, HTTP transport is layered over the SSL/TLS encrypted connection.

如果两个连接端点之间传输的信息不能被未经授权的第三方读取或篡改，则可以认为 HTTP 连接是安全的。SSL/TLS 协议是最广泛使用的技术，用于确保 HTTP 传输安全。不过，也可以使用其他加密技术。通常，HTTP 传输层位于 SSL/TLS 加密连接之上。

## 2.3 HTTP connection managers

### 2.3.1 Managed connections and connection managers

HTTP connections are complex, stateful, thread-unsafe objects which need to be properly managed to function correctly. HTTP connections can only be used by one execution thread at a time. HttpClient employs a special entity to manage access to HTTP connections called HTTP connection manager and represented by the HttpClientConnectionManager interface. The purpose of an HTTP connection manager is to serve as a factory for new HTTP connections, to manage life cycle of persistent connections and to synchronize access to persistent connections making sure that only one thread can have access to a connection at a time. Internally HTTP connection managers work with instances of ManagedHttpClientConnection acting as a proxy for a real connection that manages connection state and controls execution of I/O operations. If a managed connection is released or get explicitly closed by its consumer the underlying connection gets detached from its proxy and is returned back to the manager. Even though the service consumer still holds a reference to the proxy instance, it is no longer able to execute any I/O operations or change the state of the real connection either intentionally or unintentionally.

HTTP 连接是复杂的、有状态的、线程不安全的对象，需要正确地管理它们才能确保程序正确地工作。HTTP 连接一次只能由一个执行线程使用。HttpClient 使用一个特殊的实体来管理对 HTTP 连接的访问，该实体称为 HTTP 连接管理器，由 HttpClientConnectionManager 接口表示。HTTP 连接管理器的目的是充当新 HTTP 连接的工厂，管理持久连接的生命周期，并同步对持久连接的访问，确保一次只能有一个线程访问连接。在内部，HTTP 连接管理器使用 ManagedHttpClientConnection 实例作为一个实际连接的代理，管理连接状态并控制 I/O 操作的执行。如果托管连接被其使用者释放或显式关闭，则底层连接将从其代理分离并返回给管理器。即使服务使用者仍然持有对代理实例的引用，它也不再能够执行任何 I/O 操作，也不能更改实际连接的状态。

This is an example of acquiring a connection from a connection manager:

这是从连接管理器获取连接的例子：

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

The connection request can be terminated prematurely by calling ConnectionRequest#cancel() if necessary. This will unblock the thread blocked in the ConnectionRequest#get() method.

如果需要，可以通过调用 ConnectionRequest#cancel() 提前终止连接请求。这将解除 ConnectionRequest#get() 方法中线程的阻塞状态。

### 2.3.2 Simple connection manager

BasicHttpClientConnectionManager is a simple connection manager that maintains only one connection at a time. Even though this class is thread-safe it ought to be used by one execution thread only. BasicHttpClientConnectionManager will make an effort to reuse the connection for subsequent requests with the same route. It will, however, close the existing connection and re-open it for the given route, if the route of the persistent connection does not match that of the connection request. If the connection has been already been allocated, then java.lang.IllegalStateException is thrown.

BasicHttpClientConnectionManager 是一个简单的连接管理器，一次只维护一个连接。即使这个类是线程安全的，它也应该只被一个执行线程使用。BasicHttpClientConnectionManager 将努力重用连接，为后续请求与相同的路由。但是，如果持久连接的路由与连接请求的路由不匹配，则它将关闭现有连接并为给定路由重新打开它。如果已经分配了连接，则将 java.lang.IllegalStateException 抛出。

This connection manager implementation should be used inside an EJB container.

这个连接管理器实现应该在 EJB 容器中使用。

### 2.3.3 Pooling connection manager

PoolingHttpClientConnectionManager is a more complex implementation that manages a pool of client connections and is able to service connection requests from multiple execution threads. Connections are pooled on a per route basis. A request for a route for which the manager already has a persistent connection available in the pool will be serviced by leasing a connection from the pool rather than creating a brand new connection.

PoolingHttpClientConnectionManager 是一个更复杂的实现，它管理一个客户端连接池，能够为来自多个执行线程的连接请求提供服务。连接按每个路由汇集。对于一个路由的请求，如果管理器已经在池中拥有一个可用的持久连接，那么将通过从池中分配一个连接而不是创建一个全新的连接来满足该请求。

PoolingHttpClientConnectionManager maintains a maximum limit of connections on a per route basis and in total. Per default this implementation will create no more than 2 concurrent connections per given route and no more 20 connections in total. For many real-world applications these limits may prove too constraining, especially if they use HTTP as a transport protocol for their services.

PoolingHttpClientConnectionManager 在每个路由的基础上和总体上维护连接的最大限制。默认情况下，此实现将为每个给定路由创建不超过 2 个并发连接，并且总共不超过 20 个连接。对于许多实际应用程序来说，这些限制可能被证明过于严格，特别是如果它们将 HTTP 用作服务的传输协议时。

This example shows how the connection pool parameters can be adjusted:

这个例子展示了如何调整连接池参数：

```
PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
// Increase max total connection to 200（增加最大连接数量）
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

当不再需要 HttpClient 实例并即将超出作用域时，应关闭它的连接管理器以确保所有由管理器保持活动的连接都被关闭，并释放由这些连接分配的系统资源，这一点非常重要。

```
CloseableHttpClient httpClient = <...>
httpClient.close();
```

## 2.4 Multithreaded request execution

When equipped with a pooling connection manager such as PoolingClientConnectionManager, HttpClient can be used to execute multiple requests simultaneously using multiple threads of execution.

当配置池连接管理器（如 PoolingClientConnectionManager）时，HttpClient 可以使用多个执行线程同时执行多个请求。

The PoolingClientConnectionManager will allocate connections based on its configuration. If all connections for a given route have already been leased, a request for a connection will block until a connection is released back to the pool. One can ensure the connection manager does not block indefinitely in the connection request operation by setting 'http.conn-manager.timeout' to a positive value. If the connection request cannot be serviced within the given time period ConnectionPoolTimeoutException will be thrown.

PoolingClientConnectionManager 将根据其配置分配连接。如果已分配了给定路由的所有连接，则对该连接的请求将阻塞，直到将连接释放回池为止。通过设置 'http.conn-manager.timeout'（超时值为正值），可以确保连接管理器不会在连接请求操作中无限期阻塞。如果连接请求不能在给定的时间段内得到服务，将抛出 ConnectionPoolTimeoutException。

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

While HttpClient instances are thread safe and can be shared between multiple threads of execution, it is highly recommended that each thread maintains its own dedicated instance of HttpContext.

虽然 HttpClient 实例是线程安全的，并且可以在多个执行线程之间共享，但是强烈建议每个线程维护自己专用的 HttpContext 实例。

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

## 2.5 Connection eviction policy

One of the major shortcomings of the classic blocking I/O model is that the network socket can react to I/O events only when blocked in an I/O operation. When a connection is released back to the manager, it can be kept alive however it is unable to monitor the status of the socket and react to any I/O events. If the connection gets closed on the server side, the client side connection is unable to detect the change in the connection state (and react appropriately by closing the socket on its end).

经典阻塞 I/O 模型的一个主要缺点是，只有在 I/O 操作中阻塞时，网络 socket 才能对 I/O 事件作出反应。当一个连接被释放回管理器时，它可以保持活动状态，但是它不能监视 socket 的状态并对任何 I/O 事件作出反应。如果在服务器端关闭连接，则客户端连接无法检测连接状态中的更改（并通过关闭连接端上的 socket 做出适当的反应）。

HttpClient tries to mitigate the problem by testing whether the connection is 'stale', that is no longer valid because it was closed on the server side, prior to using the connection for executing an HTTP request. The stale connection check is not 100% reliable. The only feasible solution that does not involve a one thread per socket model for idle connections is a dedicated monitor thread used to evict connections that are considered expired due to a long period of inactivity. The monitor thread can periodically call ClientConnectionManager#closeExpiredConnections() method to close all expired connections and evict closed connections from the pool. It can also optionally call ClientConnectionManager#closeIdleConnections() method to close all connections that have been idle over a given period of time.

HttpClient 试图通过测试连接是否「陈旧」来缓解这个问题，因为在使用连接执行 HTTP 请求之前，如果连接在服务器端已经关闭，其不再有效。陈旧的连接检查不是 100% 可靠的。对于空闲连接，唯一可行的不涉及每个 socket 模型一个线程的解决方案是一个专用的监视线程，用于回收长时间不活动而被认为已过期的连接。监视器线程可以周期性地调用 ClientConnectionManager#closeExpiredConnections() 方法来关闭所有过期的连接，并从池中回收关闭的连接。它还可以选择性地调用 ClientConnectionManager#closeIdleConnections() 方法来关闭在给定时间内空闲的所有连接。

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

The HTTP specification does not specify how long a persistent connection may be and should be kept alive. Some HTTP servers use a non-standard Keep-Alive header to communicate to the client the period of time in seconds they intend to keep the connection alive on the server side. HttpClient makes use of this information if available. If the Keep-Alive header is not present in the response, HttpClient assumes the connection can be kept alive indefinitely. However, many HTTP servers in general use are configured to drop persistent connections after a certain period of inactivity in order to conserve system resources, quite often without informing the client. In case the default strategy turns out to be too optimistic, one may want to provide a custom keep-alive strategy.

HTTP 规范没有指定持久性连接可能存在多长时间，并且应该保持活动状态。一些 HTTP 服务器使用非标准的 Keep-Alive 头与客户端通信，并希望在服务器端保持连接活动的时间（以秒为单位）。如果可用，HttpClient 将使用这些信息。如果响应中没有 Keep-Alive 头，HttpClient 假定连接可以无限期地保持活动状态。然而，通常使用的许多 HTTP 服务器都被配置为在一段时间不活动之后删除持久连接，以便节省系统资源，而且常常不通知客户端。如果默认策略过于乐观，可能需要提供一个定制的 keep-alive 策略。

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

HTTP connections make use of a java.net.Socket object internally to handle transmission of data across the wire. However they rely on the ConnectionSocketFactory interface to create, initialize and connect sockets. This enables the users of HttpClient to provide application specific socket initialization code at runtime. PlainConnectionSocketFactory is the default factory for creating and initializing plain (unencrypted) sockets.

HTTP 连接在内部使用一个 java.net.Socket 对象来处理跨线路的数据传输。然而，它们依赖于 ConnectionSocketFactory 接口来创建、初始化和连接 socket。这允许 HttpClient 的用户在运行时提供特定于应用程序的 socket 初始化代码。PlainConnectionSocketFactory 是创建和初始化普通（未加密）socket 的默认工厂。

The process of creating a socket and that of connecting it to a host are decoupled, so that the socket could be closed while being blocked in the connect operation.

创建 socket 和将 socket 连接到主机的两个过程是解耦的，以便在连接操作中阻塞 socket 时可以关闭 socket。

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

LayeredConnectionSocketFactory is an extension of the ConnectionSocketFactory interface. Layered socket factories are capable of creating sockets layered over an existing plain socket. Socket layering is used primarily for creating secure sockets through proxies. HttpClient ships with SSLSocketFactory that implements SSL/TLS layering. Please note HttpClient does not use any custom encryption functionality. It is fully reliant on standard Java Cryptography (JCE) and Secure Sockets (JSEE) extensions.

LayeredConnectionSocketFactory 继承了 ConnectionSocketFactory 接口。分层 socket 工厂能够在现有的普通 socket 上创建分层的 socket。socket 分层主要用于通过代理创建安全 socket。HttpClient 附带实现 SSL/TLS 分层的 SSLSocketFactory。请注意 HttpClient 不使用任何自定义加密功能。它完全依赖于标准 Java 密码学（JCE）和安全 socket（JSEE）扩展。

### 2.7.2 Integration with connection manager

Custom connection socket factories can be associated with a particular protocol scheme as as HTTP or HTTPS and then used to create a custom connection manager.

自定义连接 socket 工厂可以与特定的协议方案（如 HTTP 或 HTTPS）相关联，然后用于创建自定义连接管理器。

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

HttpClient makes use of SSLConnectionSocketFactory to create SSL connections. SSLConnectionSocketFactory allows for a high degree of customization. It can take an instance of javax.net.ssl.SSLContext as a parameter and use it to create custom configured SSL connections.

HttpClient 使用 SSLConnectionSocketFactory 来创建 SSL 连接。SSLConnectionSocketFactory 允许高度定制。它可以将 javax.net.ssl.SSLContext 的实例作为参数，并使用它创建自定义配置的 SSL 连接。

```
KeyStore myTrustStore = <...>
SSLContext sslContext = SSLContexts.custom()
        .loadTrustMaterial(myTrustStore)
        .build();
SSLConnectionSocketFactory sslsf = new SSLConnectionSocketFactory(sslContext);
```

Customization of SSLConnectionSocketFactory implies a certain degree of familiarity with the concepts of the SSL/TLS protocol, a detailed explanation of which is out of scope for this document. Please refer to the Java™ Secure Socket Extension (JSSE) Reference Guide for a detailed description of javax.net.ssl.SSLContext and related tools.

SSLConnectionSocketFactory 的自定义意味着对 SSL/TLS 协议的概念有一定程度的熟悉，详细的解释超出了本文的范围。有关 javax.net.ssl.SSLContext 和相关工具的详细描述，请参阅 Java™ Secure Socket Extension (JSSE) Reference Guide。

### 2.7.4 Hostname verification

In addition to the trust verification and the client authentication performed on the SSL/TLS protocol level, HttpClient can optionally verify whether the target hostname matches the names stored inside the server's X.509 certificate, once the connection has been established. This verification can provide additional guarantees of authenticity of the server trust material. The javax.net.ssl.HostnameVerifier interface represents a strategy for hostname verification. HttpClient ships with two javax.net.ssl.HostnameVerifier implementations. Important: hostname verification should not be confused with SSL trust verification.

除了在 SSL/TLS 协议级别上执行的信任验证和客户端身份验证之外，一旦建立了连接，HttpClient 还可以选择性地验证目标主机名是否与存储在服务器的 X.509 证书中的名称匹配。这种验证可以为服务器信任材料的真实性提供额外的保证。接口 javax.net.ssl.HostnameVerifier 表示一种验证主机名的策略。HttpClient 附带两个 javax.net.ssl.HostnameVerifier 实现。重要提示：主机名验证不应与 SSL 信任验证混淆。

- DefaultHostnameVerifier: The default implementation used by HttpClient is expected to be compliant with RFC 2818. The hostname must match any of alternative names specified by the certificate, or in case no alternative names are given the most specific CN of the certificate subject. A wildcard can occur in the CN, and in any of the subject-alts.

DefaultHostnameVerifier：HttpClient 使用的默认实现应该与 RFC 2818 兼容。主机名必须匹配证书指定的任何替代名称，或者在没有提供证书主题的最特定 CN 的情况下。通配符可以出现在 CN 中，也可以出现在任何 subject-alts 中。

- NoopHostnameVerifier: This hostname verifier essentially turns hostname verification off. It accepts any SSL session as valid and matching the target host.

NoopHostnameVerifier：这个主机名验证器实际上关闭了主机名验证。它接受任何有效的 SSL 会话，并与目标主机匹配。

Per default HttpClient uses the DefaultHostnameVerifier implementation. One can specify a different hostname verifier implementation if desired

每个默认 HttpClient 使用「DefaultHostnameVerifier」实现。如果需要，可以指定不同的主机名验证器实现。

```
SSLContext sslContext = SSLContexts.createSystemDefault();
SSLConnectionSocketFactory sslsf = new SSLConnectionSocketFactory(sslContext, NoopHostnameVerifier.INSTANCE);
```

As of version 4.4 HttpClient uses the public suffix list kindly maintained by Mozilla Foundation to make sure that wildcards in SSL certificates cannot be misused to apply to multiple domains with a common top-level domain. HttpClient ships with a copy of the list retrieved at the time of the release. The latest revision of the list can found at https://publicsuffix.org/list/. It is highly adviseable to make a local copy of the list and download the list no more than once per day from its original location.

从 4.4 版开始，HttpClient 使用由 Mozilla 基金会维护的公共后缀列表，以确保 SSL 证书中的通配符不会被误用来应用于具有公共顶级域的多个域。HttpClient 附带了在发布时检索到的列表的副本。该列表的最新修订可以在 https://publicsuffix.org/list/ 找到。最好将列表复制到本地，每天从原始位置下载列表的次数不要超过一次。

```
PublicSuffixMatcher publicSuffixMatcher = PublicSuffixMatcherLoader
        .load(PublicSuffixMatcher.class.getResource("my-copy-effective_tld_names.dat"));
DefaultHostnameVerifier hostnameVerifier = new DefaultHostnameVerifier(publicSuffixMatcher);
```

One can disable verification against the public suffic list by using null matcher.

可以使用 null 匹配器禁用对公共后缀列表的验证。

```
DefaultHostnameVerifier hostnameVerifier = new DefaultHostnameVerifier(null);
```

## 2.8 HttpClient proxy configuration

Even though HttpClient is aware of complex routing schemes and proxy chaining, it supports only simple direct or one hop proxy connections out of the box.

尽管 HttpClient 知道复杂的路由方案和代理链接，但它只支持简单的直接或单跳代理连接。

The simplest way to tell HttpClient to connect to the target host via a proxy is by setting the default proxy parameter:

告诉 HttpClient 通过代理连接到目标主机的最简单方法是设置默认的代理参数：

```
HttpHost proxy = new HttpHost("someproxy", 8080);
DefaultProxyRoutePlanner routePlanner = new DefaultProxyRoutePlanner(proxy);
CloseableHttpClient httpclient = HttpClients.custom()
        .setRoutePlanner(routePlanner)
        .build();
```

One can also instruct HttpClient to use the standard JRE proxy selector to obtain proxy information:

还可以指示 HttpClient 使用标准的 JRE 代理选择器来获取代理信息：

```
SystemDefaultRoutePlanner routePlanner = new SystemDefaultRoutePlanner(ProxySelector.getDefault());
CloseableHttpClient httpclient = HttpClients.custom()
        .setRoutePlanner(routePlanner)
        .build();
```

Alternatively, one can provide a custom RoutePlanner implementation in order to have a complete control over the process of HTTP route computation:

或者，你可以提供一个自定义的 RoutePlanner 实现，以便完全控制 HTTP 路由计算过程：

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
