# Chapter 3. Asynchronous I/O based on NIO（基于 NIO 的异步 I/O）

Asynchronous I/O model may be more appropriate for those scenarios where raw data throughput is less important than the ability to handle thousands of simultaneous connections in a scalable, resource efficient manner. Asynchronous I/O is arguably more complex and usually requires a special care when dealing with large message payloads.

异步 I/O 模型可能更适合那些 raw data throughput is less important than the ability to handle thousands of simultaneous connections in a scalable, resource efficient manner. 异步 I/O 可能更复杂，在处理大型消息有效负载时通常需要特别小心。

## 3.1. Differences from other I/O frameworks（与其他 I/O 框架的区别）

Solves similar problems as other frameworks, but has certain distinct features:

虽然解决与其他框架类似的问题，但具备某些独有特性：

- minimalistic, optimized for data volume intensive protocols such as HTTP.

最小化，针对 HTTP 等数据量密集型协议进行了优化。

- efficient memory management: data consumer can read is only as much input data as it can process without having to allocate more memory.

高效的内存管理：数据使用者可以读取的输入数据量与它可以处理的输入数据量相等，而不需要分配更多的内存。

- direct access to the NIO channels where possible.

在可能的情况下直接访问 NIO 通道。

## 3.2. I/O reactor

HttpCore NIO is based on the Reactor pattern as described by Doug Lea. The purpose of I/O reactors is to react to I/O events and to dispatch event notifications to individual I/O sessions. The main idea of I/O reactor pattern is to break away from the one thread per connection model imposed by the classic blocking I/O model. The IOReactor interface represents an abstract object which implements the Reactor pattern. Internally, IOReactor implementations encapsulate functionality of the NIO java.nio.channels.Selector.

HttpCore NIO 基于 Doug Lea 所描述的 Reactor 设计模式。I/O Reactor 的目的是响应 I/O 事件，并向各个 I/O 会话发送事件通知。I/O Reactor 模式的主要思想是摆脱经典的阻塞 I/O 模型所强加的每个连接一个线程的模式。IOReactor 接口表示实现 Reactor 模式的抽象对象。在内部，IOReactor 实现封装了 NIO java.nio.channels.Selector 的功能。

I/O reactors usually employ a small number of dispatch threads (often as few as one) to dispatch I/O event notifications to a much greater number (often as many as several thousands) of I/O sessions or connections. It is generally recommended to have one dispatch thread per CPU core.

I/O Reactor 通常使用少量的分派线程（通常只有一个）来将 I/O 事件通知分派给更多的 I/O 会话或连接（通常多达数千个）。通常建议每个 CPU 内核有一个分派线程。

```
IOReactorConfig config = IOReactorConfig.DEFAULT;
IOReactor ioreactor = new DefaultConnectingIOReactor(config);
```

### 3.2.1. I/O dispatchers（I/O 调度）

IOReactor implementations make use of the IOEventDispatch interface to notify clients of events pending for a particular session. All methods of the IOEventDispatch are executed on a dispatch thread of the I/O reactor. Therefore, it is important that processing that takes place in the event methods will not block the dispatch thread for too long, as the I/O reactor will be unable to react to other events.

IOReactor 实现利用 IOEventDispatch 接口通知客户端特定会话的事件挂起。IOEventDispatch 的所有方法都在 I/O Reactor 的调度线程上执行。因此，重要的是，在事件方法中进行的处理不会阻塞分派线程太长时间，因为 I/O Reactor 将无法对其他事件作出反应。

```
IOReactor ioreactor = new DefaultConnectingIOReactor();

IOEventDispatch eventDispatch = <...>
ioreactor.execute(eventDispatch);
```

Generic I/O events as defined by the IOEventDispatch interface:

由 IOEventDispatch 接口定义的通用 I/O 事件：

- connected: Triggered when a new session has been created.

- inputReady: Triggered when the session has pending input.

- outputReady: Triggered when the session is ready for output.

- timeout: Triggered when the session has timed out.

- disconnected: Triggered when the session has been terminated.

### 3.2.2. I/O reactor shutdown（关闭 I/O Reactor）

The shutdown of I/O reactors is a complex process and may usually take a while to complete. I/O reactors will attempt to gracefully terminate all active I/O sessions and dispatch threads approximately within the specified grace period. If any of the I/O sessions fails to terminate correctly, the I/O reactor will forcibly shut down remaining sessions.

关闭 I/O Reactor 是一个复杂的过程，通常需要一段时间才能完成。I/O Reactor 将尝试优雅地终止所有活动 I/O 会话，并大约在指定的宽限期内分派线程。如果任何 I/O 会话未能正确终止，I/O Reactor 将强制关闭剩余的会话。

```
IOReactor ioreactor = <...>
long gracePeriod = 3000L; // milliseconds
ioreactor.shutdown(gracePeriod);
```

The IOReactor#shutdown(long) method is safe to call from any thread.

从任何线程调用 IOReactor#shutdown(long) 方法都是安全的。

### 3.2.3. I/O sessions

The IOSession interface represents a sequence of logically related data exchanges between two end points. IOSession encapsulates functionality of NIO java.nio.channels.SelectionKey and java.nio.channels.SocketChannel. The channel associated with the IOSession can be used to read data from and write data to the session.

IOSession 接口表示两个端点之间逻辑相关的数据交换序列。IOSession 封装了 NIO java.nio.channels.SelectionKey 和 java.nio.channels.SocketChannel 的功能。与 IOSession 关联的通道可用于从会话中读取数据并将数据写入会话。

```
IOSession iosession = <...>
ReadableByteChannel ch = (ReadableByteChannel) iosession.channel();
ByteBuffer dst = ByteBuffer.allocate(2048);
ch.read(dst);
```

### 3.2.4. I/O session state management（I/O 会话状态管理）

I/O sessions are not bound to an execution thread, therefore one cannot use the context of the thread to store a session's state. All details about a particular session must be stored within the session itself.

I/O 会话不绑定到执行线程，因此不能使用线程的上下文来存储会话的状态。有关特定会话的所有详细信息必须存储在会话本身中。

```
IOSession iosession = <...>
Object someState = <...>
iosession.setAttribute("state", someState);
...
IOSession iosession = <...>
Object currentState = iosession.getAttribute("state");
```

Please note that if several sessions make use of shared objects, access to those objects must be made thread-safe.

请注意，如果多个会话使用了共享对象，那么对这些对象的访问必须是线程安全的。

### 3.2.5. I/O session event mask

One can declare an interest in a particular type of I/O events for a particular I/O session by setting its event mask.

可以通过设置特定 I/O 会话的事件掩码来声明对特定类型 I/O 事件的意图。

```
IOSession iosession = <...>
iosession.setEventMask(SelectionKey.OP_READ | SelectionKey.OP_WRITE);
```

One can also toggle OP_READ and OP_WRITE flags individually.

还可以分别切换 OP_READ 和 OP_WRITE 标志。

```
IOSession iosession = <...>
iosession.setEvent(SelectionKey.OP_READ);
iosession.clearEvent(SelectionKey.OP_READ);
```

Event notifications will not take place if the corresponding interest flag is not set.

如果没有设置相应的意图标志，则不会发生事件通知。

### 3.2.6. I/O session buffers

Quite often I/O sessions need to maintain internal I/O buffers in order to transform input / output data prior to returning it to the consumer or writing it to the underlying channel. Memory management in HttpCore NIO is based on the fundamental principle that the data a consumer can read, is only as much input data as it can process without having to allocate more memory. That means, quite often some input data may remain unread in one of the internal or external session buffers. The I/O reactor can query the status of these session buffers, and make sure the consumer gets notified correctly as more data gets stored in one of the session buffers, thus allowing the consumer to read the remaining data once it is able to process it. I/O sessions can be made aware of the status of external session buffers using the SessionBufferStatus interface.

通常，I/O 会话需要维护内部 I/O 缓冲区，以便在将输入/输出数据返回给使用者或将其写入底层通道之前对其进行转换。HttpCore NIO 中的内存管理基于这样一个基本原则：使用者可以读取的数据是其能够处理的输入数据，而不必分配更多的内存。这意味着，在一个内部或外部会话缓冲区中，一些输入数据可能经常保持未读状态。I/O reactor 可以查询这些会话缓冲区的状态，并确保在一个会话缓冲区中存储更多数据时正确地通知使用者，从而允许使用者在能够处理剩余数据时读取剩余数据。可以使用 SessionBufferStatus 接口使 I/O 会话掌握外部会话缓冲区的状态。

```
IOSession iosession = <...>
SessionBufferStatus myBufferStatus = <...>
iosession.setBufferStatus(myBufferStatus);
iosession.hasBufferedInput();
iosession.hasBufferedOutput();
```

### 3.2.7. I/O session shutdown

One can close an I/O session gracefully by calling IOSession#close() allowing the session to be closed in an orderly manner or by calling IOSession#shutdown() to forcibly close the underlying channel. The distinction between two methods is of primary importance for those types of I/O sessions that involve some sort of a session termination handshake such as SSL/TLS connections.

可以通过调用 IOSession#close() 或调用 IOSession#shutdown() 强制关闭底层通道来优雅地关闭 I/O 会话。对于那些涉及某种会话终止握手（如 SSL/TLS 连接）的 I/O 会话类型，两种方法之间的区别至关重要。

### 3.2.8. Listening I/O reactors

ListeningIOReactor represents an I/O reactor capable of listening for incoming connections on one or several ports.

ListeningIOReactor 表示能够监听一个或多个端口上传入连接的 I/O reactor。

```
ListeningIOReactor ioreactor = <...>
ListenerEndpoint ep1 = ioreactor.listen(new InetSocketAddress(8081) );
ListenerEndpoint ep2 = ioreactor.listen(new InetSocketAddress(8082));
ListenerEndpoint ep3 = ioreactor.listen(new InetSocketAddress(8083));
// Wait until all endpoints are up
ep1.waitFor();
ep2.waitFor();
ep3.waitFor();
```

Once an endpoint is fully initialized it starts accepting incoming connections and propagates I/O activity notifications to the IOEventDispatch instance.

一旦端点被完全初始化，它就开始接受传入的连接，并将 I/O 活动通知传播到 IOEventDispatch 实例。

One can obtain a set of registered endpoints at runtime, query the status of an endpoint at runtime, and close it if desired.

可以在运行时获得一组已注册的端点，在运行时查询端点的状态，并在需要时关闭它。

```
ListeningIOReactor ioreactor = <...>

Set<ListenerEndpoint> eps = ioreactor.getEndpoints();
for (ListenerEndpoint ep: eps) {
    // Still active?
    System.out.println(ep.getAddress());
    if (ep.isClosed()) {
        // If not, has it terminated due to an exception?
        if (ep.getException() != null) {
            ep.getException().printStackTrace();
        }
    } else {
        ep.close();
    }
}
```

### 3.2.9. Connecting I/O reactors

ConnectingIOReactor represents an I/O reactor capable of establishing connections with remote hosts.

ConnectingIOReactor 表示能够与远程主机建立连接的 I/O reactor。

```
ConnectingIOReactor ioreactor = <...>

SessionRequest sessionRequest = ioreactor.connect(
        new InetSocketAddress("www.google.com", 80),
        null, null, null);
```

Opening a connection to a remote host usually tends to be a time consuming process and may take a while to complete. One can monitor and control the process of session initialization by means of the SessionRequestinterface.

打开到远程主机的连接通常是一个耗时的过程，可能需要一段时间才能完成。可以通过 SessionRequestinterface 监视和控制会话初始化的过程。

```
// Make sure the request times out if connection
// has not been established after 1 sec
sessionRequest.setConnectTimeout(1000);
// Wait for the request to complete
sessionRequest.waitFor();
// Has request terminated due to an exception?
if (sessionRequest.getException() != null) {
    sessionRequest.getException().printStackTrace();
}
// Get hold of the new I/O session
IOSession iosession = sessionRequest.getSession();
```

SessionRequest implementations are expected to be thread-safe. Session request can be aborted at any time by calling IOSession#cancel() from another thread of execution.

SessionRequest 实现应该是线程安全的。通过从另一个执行线程调用 IOSession#cancel()，可以在任何时候中止会话请求。

```
if (!sessionRequest.isCompleted()) {
    sessionRequest.cancel();
}
```

One can pass several optional parameters to the ConnectingIOReactor#connect() method to exert a greater control over the process of session initialization.

可以将几个可选参数传递给 ConnectingIOReactor#connect() 方法，以便对会话初始化过程施加更大的控制。

A non-null local socket address parameter can be used to bind the socket to a specific local address.

non-null 本地套接字地址参数可用于将套接字绑定到特定的本地地址。

```
ConnectingIOReactor ioreactor = <...>

SessionRequest sessionRequest = ioreactor.connect(
        new InetSocketAddress("www.google.com", 80),
        new InetSocketAddress("192.168.0.10", 1234),
        null, null);
```

One can provide an attachment object, which will be added to the new session's context upon initialization. This object can be used to pass an initial processing state to the protocol handler.

可以提供一个附件对象，该对象将在初始化时添加到新会话的上下文中。此对象可用于将初始处理状态传递给协议处理程序。

```
SessionRequest sessionRequest = ioreactor.connect(
        new InetSocketAddress("www.google.com", 80),
        null, new HttpHost("www.google.ru"), null);

IOSession iosession = sessionRequest.getSession();
HttpHost virtualHost = (HttpHost) iosession.getAttribute(
    IOSession.ATTACHMENT_KEY);
```

It is often desirable to be able to react to the completion of a session request asynchronously without having to wait for it, blocking the current thread of execution. One can optionally provide an implementation SessionRequestCallback interface to get notified of events related to session requests, such as request completion, cancellation, failure or timeout.

通常希望能够对会话请求的完成做出异步响应，而不必等待它，导致阻塞当前的执行线程。可以选择提供一个实现 SessionRequestCallback 接口来获得与会话请求相关的事件通知，例如请求完成、取消、失败或超时。

```
ConnectingIOReactor ioreactor = <...>

SessionRequest sessionRequest = ioreactor.connect(
        new InetSocketAddress("www.google.com", 80), null, null,
        new SessionRequestCallback() {

            public void cancelled(SessionRequest request) {
            }

            public void completed(SessionRequest request) {
                System.out.println("new connection to " +
                    request.getRemoteAddress());
            }

            public void failed(SessionRequest request) {
                if (request.getException() != null) {
                    request.getException().printStackTrace();
                }
            }

            public void timeout(SessionRequest request) {
            }

        });
```

## 3.3. I/O reactor configuration

I/O reactors by default use system dependent configuration which in most cases should be sensible enough.

I/O reactor 默认使用与系统相关的配置，在大多数情况下，这种配置应该足够合理。

```
IOReactorConfig config = IOReactorConfig.DEFAULT;
IOReactor ioreactor = new DefaultListeningIOReactor(config);
```

However in some cases custom settings may be necessary, for instance, in order to alter default socket properties and timeout values. One should rarely need to change other parameters.

```
IOReactorConfig config = IOReactorConfig.custom()
        .setTcpNoDelay(true)
        .setSoTimeout(5000)
        .setSoReuseAddress(true)
        .setConnectTimeout(5000)
        .build();
IOReactor ioreactor = new DefaultListeningIOReactor(config);
```

### 3.3.1. Queuing of I/O interest set operations

Several older JRE implementations (primarily from IBM) include what Java API documentation refers to as a naive implementation of the java.nio.channels.SelectionKey class. The problem with java.nio.channels.SelectionKey in such JREs is that reading or writing of the I/O interest set may block indefinitely if the I/O selector is in the process of executing a select operation. HttpCore NIO can be configured to operate in a special mode wherein I/O interest set operations are queued and executed by on the dispatch thread only when the I/O selector is not engaged in a select operation.

几个较早的 JRE 实现（主要来自 IBM）包括 Java API 文档中提到的 java.nio.channels.SelectionKey 类的简单实现。在 JRE 中，java.nio.channels.SelectionKey 的问题是，writing of the I/O interest set may block indefinitely if the I/O selector is in the process of executing a select operation. 可以将 HttpCore NIO 配置为以一种特殊的模式进行操作，在这种模式中，只有当 I/O 选择器不参与 select 操作时，I/O interest set operations are queued and executed by on the dispatch thread only。

```
IOReactorConfig config = IOReactorConfig.custom()
        .setInterestOpQueued(true)
        .build();
```

## 3.4. I/O reactor exception handling

Protocol specific exceptions as well as those I/O exceptions thrown in the course of interaction with the session's channel are to be expected and are to be dealt with by specific protocol handlers. These exceptions may result in termination of an individual session but should not affect the I/O reactor and all other active sessions. There are situations, however, when the I/O reactor itself encounters an internal problem such as an I/O exception in the underlying NIO classes or an unhandled runtime exception. Those types of exceptions are usually fatal and will cause the I/O reactor to shut down automatically.

特定于协议的异常以及在与会话通道交互过程中抛出的 I/O 异常都是预期的，并由特定的协议处理程序处理。这些异常可能导致单个会话的终止，但不应影响 I/O reactor 和所有其他活动会话。但是，在某些情况下，当 I/O reactor 本身遇到内部问题时，例如底层 NIO 类中的 I/O 异常或未处理的运行时异常。这些类型的异常通常是致命的，会导致 I/O reactor 自动关闭。

There is a possibility to override this behavior and prevent I/O reactors from shutting down automatically in case of a runtime exception or an I/O exception in internal classes. This can be accomplished by providing a custom implementation of the IOReactorExceptionHandler interface.

在运行时异常或内部类中的 I/O 异常情况下，可以覆盖此行为并防止 I/O reactor 自动关闭。这可以通过提供 IOReactorExceptionHandler 接口的自定义实现来实现。

```
DefaultConnectingIOReactor ioreactor = <...>

ioreactor.setExceptionHandler(new IOReactorExceptionHandler() {

    public boolean handle(IOException ex) {
        if (ex instanceof BindException) {
            // bind failures considered OK to ignore
            return true;
        }
        return false;
    }

    public boolean handle(RuntimeException ex) {
        if (ex instanceof UnsupportedOperationException) {
            // Unsupported operations considered OK to ignore
            return true;
        }
        return false;
    }

});
```

One needs to be very careful about discarding exceptions indiscriminately. It is often much better to let the I/O reactor shut down itself cleanly and restart it rather than leaving it in an inconsistent or unstable state.

对于不加选择地丢弃异常，需要非常小心。通常，让 I/O reactor 干净地关闭并重新启动比让它处于不一致或不稳定的状态要好得多。

### 3.4.1. I/O reactor audit log

If an I/O reactor is unable to automatically recover from an I/O or a runtime exception it will enter the shutdown mode. First off, it will close all active listeners and cancel all pending new session requests. Then it will attempt to close all active I/O sessions gracefully giving them some time to flush pending output data and terminate cleanly. Lastly, it will forcibly shut down those I/O sessions that still remain active after the grace period. This is a fairly complex process, where many things can fail at the same time and many different exceptions can be thrown in the course of the shutdown process. The I/O reactor will record all exceptions thrown during the shutdown process, including the original one that actually caused the shutdown in the first place, in an audit log. One can examine the audit log and decide whether it is safe to restart the I/O reactor.

如果 I/O reactor 无法从 I/O 或运行时异常中自动恢复，它将进入关闭模式。首先，它将关闭所有活动侦听器并取消所有挂起的新会话请求。然后，它将尝试优雅地关闭所有活动 I/O 会话，给它们一些时间来刷新挂起的输出数据并干净地终止。最后，它将强制关闭那些在宽限期之后仍然活跃的 I/O 会话。这是一个相当复杂的过程，许多事情可能同时失败，在关闭过程中可能抛出许多不同的异常。I/O reactor 将在日志中记录在关闭过程中抛出的所有异常，包括最初导致关闭的那个异常。您可以检查日志，并确定重新启动 I/O reactor 是否安全。

```
DefaultConnectingIOReactor ioreactor = <...>

// Give it 5 sec grace period
ioreactor.shutdown(5000);
List<ExceptionEvent> events = ioreactor.getAuditLog();
for (ExceptionEvent event: events) {
    System.err.println("Time: " + event.getTimestamp());
    event.getCause().printStackTrace();
}
```

## 3.5. Non-blocking HTTP connections

Effectively non-blocking HTTP connections are wrappers around IOSession with HTTP specific functionality. Non-blocking HTTP connections are stateful and not thread-safe. Input / output operations on non-blocking HTTP connections should be restricted to the dispatch events triggered by the I/O event dispatch thread.

有效的非阻塞 HTTP 连接是围绕 IOSession 的包装器，具有 HTTP 特定的功能。非阻塞 HTTP 连接是有状态的，不是线程安全的。非阻塞 HTTP 连接上的输入/输出操作应该限制在 I/O 事件分派线程触发的分派事件范围内。

### 3.5.1. Execution context of non-blocking HTTP connections

Non-blocking HTTP connections are not bound to a particular thread of execution and therefore they need to maintain their own execution context. Each non-blocking HTTP connection has an HttpContext instance associated with it, which can be used to maintain a processing state. The HttpContext instance is thread-safe and can be manipulated from multiple threads.

非阻塞 HTTP 连接不绑定到特定的执行线程，因此它们需要维护自己的执行上下文。每个非阻塞 HTTP 连接都有一个与之关联的 HttpContext 实例，该实例可用于维护处理状态。HttpContext 实例是线程安全的，可以从多个线程操作。

```
DefaultNHttpClientConnection conn = <...>
Object myStateObject = <...>

HttpContext context = conn.getContext();
context.setAttribute("state", myStateObject);
```

### 3.5.2. Working with non-blocking HTTP connections

At any point of time one can obtain the request and response objects currently being transferred over the non-blocking HTTP connection. Any of these objects, or both, can be null if there is no incoming or outgoing message currently being transferred.

在任何时候，都可以通过非阻塞 HTTP 连接获得当前正在传输的请求和响应对象。如果当前没有传输传入或传出消息，则这些对象中的任何一个（或两个）都可以为空。

```
NHttpConnection conn = <...>

HttpRequest request = conn.getHttpRequest();
if (request != null) {
    System.out.println("Transferring request: " +
            request.getRequestLine());
}
HttpResponse response = conn.getHttpResponse();
if (response != null) {
    System.out.println("Transferring response: " +
            response.getStatusLine());
}
```

However, please note that the current request and the current response may not necessarily represent the same message exchange! Non-blocking HTTP connections can operate in a full duplex mode. One can process incoming and outgoing messages completely independently from one another. This makes non-blocking HTTP connections fully pipelining capable, but at same time implies that this is the job of the protocol handler to match logically related request and the response messages.

但是，请注意，当前请求和当前响应不一定代表相同的消息交换！非阻塞 HTTP 连接可以在全双工模式下运行。可以完全独立地处理传入和传出的消息。这使得非阻塞 HTTP 连接完全能够实现流水线，但同时也意味着这是协议处理程序的工作，以匹配逻辑相关的请求和响应消息。

Over-simplified process of submitting a request on the client side may look like this:

在客户端提交请求的简化过程可能是这样的：

```
NHttpClientConnection conn = <...>
// Obtain execution context
HttpContext context = conn.getContext();
// Obtain processing state
Object state = context.getAttribute("state");
// Generate a request based on the state information
HttpRequest request = new BasicHttpRequest("GET", "/");

conn.submitRequest(request);
System.out.println(conn.isRequestSubmitted());
```

Over-simplified process of submitting a response on the server side may look like this:

在服务器端提交请求的简化过程可能是这样的：

```
NHttpServerConnection conn = <...>
// Obtain execution context
HttpContext context = conn.getContext();
// Obtain processing state
Object state = context.getAttribute("state");

// Generate a response based on the state information
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1,
    HttpStatus.SC_OK, "OK");
BasicHttpEntity entity = new BasicHttpEntity();
entity.setContentType("text/plain");
entity.setChunked(true);
response.setEntity(entity);

conn.submitResponse(response);
System.out.println(conn.isResponseSubmitted());
```

Please note that one should rarely need to transmit messages using these low level methods and should use appropriate higher level HTTP service implementations instead.

请注意，很少需要使用这些底层方法来传输消息，而应该使用适当的高级 HTTP 服务实现。

### 3.5.3. HTTP I/O control

All non-blocking HTTP connections classes implement IOControl interface, which represents a subset of connection functionality for controlling interest in I/O even notifications. IOControl instances are expected to be fully thread-safe. Therefore IOControl can be used to request / suspend I/O event notifications from any thread.

所有非阻塞的 HTTP 连接类都实现了 IOControl 接口，它表示控制 I/O 甚至通知的连接功能的子集。IOControl 实例应该是完全线程安全的。因此，IOControl 可以用于从任何线程请求/挂起 I/O 事件通知。

One must take special precautions when interacting with non-blocking connections. HttpRequest and HttpResponse are not thread-safe. It is generally advisable that all input / output operations on a non-blocking connection are executed from the I/O event dispatch thread.

在与非阻塞连接交互时，必须采取特殊的预防措施。HttpRequest 和 HttpResponse 不是线程安全的。通常建议非阻塞连接上的所有输入/输出操作都从 I/O 事件调度线程执行。

The following pattern is recommended:

建议采用以下模式：

- Use IOControl interface to pass control over connection's I/O events to another thread / session.

使用 IOControl 接口将连接的 I/O 事件的控制权传递给另一个线程/会话。

- If input / output operations need be executed on that particular connection, store all the required information (state) in the connection context and request the appropriate I/O operation by calling IOControl#requestInput() or IOControl#requestOutput() method.

如果需要在该连接上执行输入/输出操作，请将所有需要的信息（state）存储在连接上下文中，并通过调用 IOControl#requestInput() 或 IOControl#requestOutput() 方法请求适当的 I/O 操作。

- Execute the required operations from the event method on the dispatch thread using information stored in connection context.

使用存储在连接上下文中的信息在分派线程上的 event 方法中执行所需的操作。

Please note all operations that take place in the event methods should not block for too long, because while the dispatch thread remains blocked in one session, it is unable to process events for all other sessions. I/O operations with the underlying channel of the session are not a problem as they are guaranteed to be non-blocking.

请注意，在事件方法中发生的所有操作不应该阻塞太长时间，因为当分派线程在一个会话中保持阻塞时，它无法处理所有其他会话的事件。使用会话的底层通道的 I/O 操作不是问题，因为它们保证是非阻塞的。

### 3.5.4. Non-blocking content transfer

The process of content transfer for non-blocking connections works completely differently compared to that of blocking connections, as non-blocking connections need to accommodate to the asynchronous nature of the NIO model. The main distinction between two types of connections is inability to use the usual, but inherently blocking java.io.InputStream and java.io.OutputStream classes to represent streams of inbound and outbound content. HttpCore NIO provides ContentEncoder and ContentDecoder interfaces to handle the process of asynchronous content transfer. Non-blocking HTTP connections will instantiate the appropriate implementation of a content codec based on properties of the entity enclosed with the message.

非阻塞连接的内容传输过程与阻塞连接的工作方式完全不同，因为非阻塞连接需要适应 NIO 模型的异步特性。两种连接类型之间的主要区别是不能使用通常的、但本质上阻塞了 java.io.InputStream 和 java.io.OutputStream 类来表示入站和出站内容流。HttpCore NIO 提供了 ContentEncoder 和 ContentDecoder 接口来处理异步内容传输的过程。非阻塞 HTTP 连接将基于消息所包含的实体的属性实例化内容编解码器的适当实现。

Non-blocking HTTP connections will fire input events until the content entity is fully transferred.

非阻塞 HTTP 连接将触发输入事件，直到内容实体被完全传输。

```
ContentDecoder decoder = <...>
//Read data in
ByteBuffer dst = ByteBuffer.allocate(2048);
decoder.read(dst);
// Decode will be marked as complete when
// the content entity is fully transferred
if (decoder.isCompleted()) {
    // Done
}
```

Non-blocking HTTP connections will fire output events until the content entity is marked as fully transferred.

非阻塞 HTTP 连接将触发输出事件，直到内容实体被标记为完全传输为止。

```
ContentEncoder encoder = <...>
// Prepare output data
ByteBuffer src = ByteBuffer.allocate(2048);
// Write data out
encoder.write(src);
// Mark content entity as fully transferred when done
encoder.complete();
```

Please note, one still has to provide an HttpEntity instance when submitting an entity enclosing message to the non-blocking HTTP connection. Properties of that entity will be used to initialize an ContentEncoder instance to be used for transferring entity content. Non-blocking HTTP connections, however, ignore inherently blocking HttpEntity#getContent() and HttpEntity#writeTo() methods of the enclosed entities.

请注意，在向非阻塞 HTTP 连接提交包含消息的实体时，仍然需要提供 HttpEntity 实例。该实体的属性将用于初始化一个 ContentEncoder 实例，用于传输实体内容。但是，非阻塞的 HTTP 连接忽略了被阻塞的 HttpEntity#getContent() 和 HttpEntity#writeTo() 方法。

```
NHttpServerConnection conn  = <...>

HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1,
    HttpStatus.SC_OK, "OK");
BasicHttpEntity entity = new BasicHttpEntity();
entity.setContentType("text/plain");
entity.setChunked(true);
entity.setContent(null);
response.setEntity(entity);

conn.submitResponse(response);
```

Likewise, incoming entity enclosing message will have an HttpEntity instance associated with them, but an attempt to call HttpEntity#getContent() or HttpEntity#writeTo() methods will cause an java.lang.IllegalStateException. The HttpEntity instance can be used to determine properties of the incoming entity such as content length.

类似地，包含消息的传入实体将有一个与之关联的 HttpEntity 实例，但是试图调用 HttpEntity#getContent() 或 HttpEntity#writeTo() 方法将导致 java.lang.IllegalStateException。HttpEntity 实例可用于确定传入实体的属性，如内容长度。

```
NHttpClientConnection conn = <...>

HttpResponse response = conn.getHttpResponse();
HttpEntity entity = response.getEntity();
if (entity != null) {
    System.out.println(entity.getContentType());
    System.out.println(entity.getContentLength());
    System.out.println(entity.isChunked());
}
```

### 3.5.5. Supported non-blocking content transfer mechanisms

Default implementations of the non-blocking HTTP connection interfaces support three content transfer mechanisms defined by the HTTP/1.1 specification:

非阻塞 HTTP 连接接口的默认实现支持 HTTP/1.1 规范定义的三种内容传输机制：

- Content-Length **delimited**: The end of the content entity is determined by the value of the Content-Length header. Maximum entity length: Long#MAX_VALUE.

- **Identity coding**: The end of the content entity is demarcated by closing the underlying connection (end of stream condition). For obvious reasons the identity encoding can only be used on the server side. Max entity length: unlimited.

- **Chunk coding**: The content is sent in small chunks. Max entity length: unlimited.

The appropriate content codec will be created automatically depending on properties of the entity enclosed with the message.

适当的内容编解码器将根据消息所包含的实体的属性自动创建。

### 3.5.6. Direct channel I/O

Content codes are optimized to read data directly from or write data directly to the underlying I/O session's channel, whenever possible avoiding intermediate buffering in a session buffer. Moreover, those codecs that do not perform any content transformation (Content-Length delimited and identity codecs, for example) can leverage NIO java.nio.FileChannel methods for significantly improved performance of file transfer operations both inbound and outbound.

内容代码被优化为直接从底层 I/O 会话通道读取数据或直接将数据写入底层 I/O 会话通道，尽可能避免会话缓冲区中的中间缓冲。而且，那些不执行任何内容转换的编解码器（例如，内容长度分隔的编解码器和标识编解码器）可以利用 NIO java.nio.FileChannel 方法来显著提高入站和出站文件传输操作的性能。

If the actual content decoder implements FileContentDecoder one can make use of its methods to read incoming content directly to a file bypassing an intermediate java.nio.ByteBuffer.

如果实际的内容解码器实现了 FileContentDecoder，那么就可以利用它的方法直接将传入的内容读入文件，绕过中间的 java.nio.ByteBuffer。

```
ContentDecoder decoder = <...>
//Prepare file channel
FileChannel dst;
//Make use of direct file I/O if possible
if (decoder instanceof FileContentDecoder) {
    long Bytesread = ((FileContentDecoder) decoder)
        .transfer(dst, 0, 2048);
     // Decode will be marked as complete when
     // the content entity is fully transmitted
     if (decoder.isCompleted()) {
         // Done
     }
}
```

If the actual content encoder implements FileContentEncoder one can make use of its methods to write outgoing content directly from a file bypassing an intermediate java.nio.ByteBuffer.

如果实际的内容编码器实现了 FileContentEncoder，那么就可以利用它的方法直接从文件中编写输出内容，绕过中间的 java.nio.ByteBuffer。

```
ContentEncoder encoder = <...>
// Prepare file channel
FileChannel src;
// Make use of direct file I/O if possible
if (encoder instanceof FileContentEncoder) {
    // Write data out
    long bytesWritten = ((FileContentEncoder) encoder)
        .transfer(src, 0, 2048);
    // Mark content entity as fully transferred when done
    encoder.complete();
}
```

## 3.6. HTTP I/O event dispatchers

HTTP I/O event dispatchers serve to convert generic I/O events triggered by an I/O reactor to HTTP protocol specific events. They rely on NHttpClientEventHandler and NHttpServerEventHandler interfaces to propagate HTTP protocol events to a HTTP protocol handler.

HTTP I/O 事件分发器用于将 I/O reactor 触发的一般 I/O 事件转换为 HTTP 协议特定的事件。它们依赖于 NHttpClientEventHandler 和 NHttpServerEventHandler 接口将 HTTP 协议事件传播到 HTTP 协议处理程序。

Server side HTTP I/O events as defined by the NHttpServerEventHandler interface:

由 NHttpServerEventHandler 接口定义的服务器端 HTTP I/O 事件：

- connected: Triggered when a new incoming connection has been created.

connected：在创建新的传入连接时触发。

- requestReceived: Triggered when a new HTTP request is received. The connection passed as a parameter to this method is guaranteed to return a valid HTTP request object. If the request received encloses a request entity this method will be followed a series of inputReady events to transfer the request content.

requestReceived：当接收到新的 HTTP 请求时触发。作为参数传递给这个方法的连接保证返回一个有效的 HTTP 请求对象。如果接收到的请求包含一个请求实体，则该方法将遵循一系列 inputReady 事件来传输请求内容。

- inputReady: Triggered when the underlying channel is ready for reading a new portion of the request entity through the corresponding content decoder. If the content consumer is unable to process the incoming content, input event notifications can temporarily suspended using IOControl interface (super interface of NHttpServerConnection). Please note that the NHttpServerConnection and ContentDecoder objects are not thread-safe and should only be used within the context of this method call. The IOControl object can be shared and used on other thread to resume input event notifications when the handler is capable of processing more content.

inputReady：当底层通道准备通过相应的内容解码器读取请求实体的新部分时触发。如果内容使用者无法处理传入的内容，则可以使用 IOControl 接口（NHttpServerConnection 的超级接口）临时暂停输入事件通知。请注意，NHttpServerConnection 和 ContentDecoder 对象不是线程安全的，应该只在这个方法调用的上下文中使用。当处理程序能够处理更多内容时，可以在其他线程上共享和使用 IOControl 对象来恢复输入事件通知。

- responseReady: Triggered when the connection is ready to accept new HTTP response. The protocol handler does not have to submit a response if it is not ready.

responseReady：当连接准备接受新的 HTTP 响应时触发。如果还没有准备好，协议处理程序不必提交响应。

- outputReady: Triggered when the underlying channel is ready for writing a next portion of the response entity through the corresponding content encoder. If the content producer is unable to generate the outgoing content, output event notifications can be temporarily suspended using IOControl interface (super interface of NHttpServerConnection). Please note that the NHttpServerConnection and ContentEncoder objects are not thread-safe and should only be used within the context of this method call. The IOControl object can be shared and used on other thread to resume output event notifications when more content is made available.

outputReady：当底层通道准备好通过相应的内容编码器写入响应实体的下一部分时触发。如果内容生产者无法生成输出内容，可以使用 IOControl 接口（NHttpServerConnection 的超级接口）临时暂停输出事件通知。请注意，NHttpServerConnection 和 ContentEncoder 对象不是线程安全的，应该只在这个方法调用的上下文中使用。IOControl 对象可以在其他线程上共享和使用，以便在更多内容可用时恢复输出事件通知。

- exception: Triggered when an I/O error occurrs while reading from or writing to the underlying channel or when an HTTP protocol violation occurs while receiving an HTTP request.

exception：当从底层通道读写数据时发生 I/O 错误，或者当接收 HTTP 请求时发生 HTTP 协议冲突时触发。

- timeout: Triggered when no input is detected on this connection over the maximum period of inactivity.

timeout：在最大不活动期间，此连接上未检测到输入时触发。

- closed: Triggered when the connection has been closed.

closed：当连接关闭时触发。

Client side HTTP I/O events as defined by the NHttpClientEventHandler interface:

由 NHttpClientEventHandler 接口定义的客户端 HTTP I/O 事件：

- connected: Triggered when a new outgoing connection has been created. The attachment object passed as a parameter to this event is an arbitrary object that was attached to the session request.

connected：在创建新的传出连接时触发。作为参数传递到此事件的附件对象是附加到会话请求的任意对象。

- requestReady: Triggered when the connection is ready to accept new HTTP request. The protocol handler does not have to submit a request if it is not ready.

requestReady：当连接准备接受新的 HTTP 请求时触发。如果没有准备好，协议处理程序不必提交请求。

- outputReady: Triggered when the underlying channel is ready for writing a next portion of the request entity through the corresponding content encoder. If the content producer is unable to generate the outgoing content, output event notifications can be temporarily suspended using IOControl interface (super interface of NHttpClientConnection). Please note that the NHttpClientConnection and ContentEncoder objects are not thread-safe and should only be used within the context of this method call. The IOControl object can be shared and used on other thread to resume output event notifications when more content is made available.

outputReady：当底层通道准备好通过相应的内容编码器写入请求实体的下一部分时触发。如果内容生产者无法生成输出内容，可以使用 IOControl 接口（NHttpClientConnection 的超级接口）临时暂停输出事件通知。请注意，NHttpClientConnection 和 ContentEncoder 对象不是线程安全的，应该只在这个方法调用的上下文中使用。IOControl 对象可以在其他线程上共享和使用，以便在更多内容可用时恢复输出事件通知。

- responseReceived: Triggered when an HTTP response is received. The connection passed as a parameter to this method is guaranteed to return a valid HTTP response object. If the response received encloses a response entity this method will be followed a series of inputReady events to transfer the response content.

responseReceived：当收到 HTTP 响应时触发。作为参数传递给这个方法的连接保证返回一个有效的 HTTP 响应对象。如果接收到的响应包含一个响应实体，则该方法将遵循一系列 inputReady 事件来传输响应内容。

- inputReady: Triggered when the underlying channel is ready for reading a new portion of the response entity through the corresponding content decoder. If the content consumer is unable to process the incoming content, input event notifications can be temporarily suspended using IOControl interface (super interface of NHttpClientConnection). Please note that the NHttpClientConnection and ContentDecoder objects are not thread-safe and should only be used within the context of this method call. The IOControl object can be shared and used on other thread to resume input event notifications when the handler is capable of processing more content.

inputReady：当底层通道准备好通过相应的内容解码器读取响应实体的新部分时触发。如果内容使用者无法处理传入的内容，可以使用 IOControl 接口（NHttpClientConnection 的超级接口）临时暂停输入事件通知。请注意，NHttpClientConnection 和 ContentDecoder 对象不是线程安全的，应该只在这个方法调用的上下文中使用。当处理程序能够处理更多内容时，可以在其他线程上共享和使用 IOControl 对象来恢复输入事件通知。

- exception: Triggered when an I/O error occurs while reading from or writing to the underlying channel or when an HTTP protocol violation occurs while receiving an HTTP response.

exception：当从底层通道读取或写入数据时发生 I/O 错误，或者在接收 HTTP 响应时发生 HTTP 协议冲突时触发。

- timeout: Triggered when no input is detected on this connection over the maximum period of inactivity.

- closed: Triggered when the connection has been closed.

## 3.7. Non-blocking HTTP content producers

As discussed previously the process of content transfer for non-blocking connections works completely differently compared to that for blocking connections. For obvious reasons classic I/O abstraction based on inherently blocking java.io.InputStream and java.io.OutputStream classes is not well suited for asynchronous data transfer. In order to avoid inefficient and potentially blocking I/O operation redirection through java.nio.channels.Channles#newChannel non-blocking HTTP entities are expected to implement NIO specific extension interface HttpAsyncContentProducer.

如前所述，非阻塞连接的内容传输过程与阻塞连接的工作方式完全不同。显然，基于固有的阻塞 java.io.InputStream 和 java.io.OutputStream 类的经典 I/O 抽象并不适合异步数据传输。为了避免低效和潜在的阻塞 I/O 操作重定向通过 java.nio.channels.Channles#newChannel 非阻塞的 HTTP 实体被期望实现 NIO 特定的扩展接口 HttpAsyncContentProducer。

The HttpAsyncContentProducer interface defines several additional method for efficient streaming of content to a non-blocking HTTP connection:

HttpAsyncContentProducer 接口定义了几个额外的方法来高效地将内容流送到一个非阻塞的 HTTP 连接：

- produceContent: Invoked to write out a chunk of content to the ContentEncoder . The IOControl interface can be used to suspend output events if the entity is temporarily unable to produce more content. When all content is finished, the producer MUST call ContentEncoder#complete(). Failure to do so may cause the entity to be incorrectly delimited. Please note that the ContentEncoder object is not thread-safe and should only be used within the context of this method call. The IOControl object can be shared and used on other thread resume output event notifications when more content is made available.

produceContent：被调用来向 ContentEncoder 写出一大块内容。如果实体暂时无法生成更多内容，可以使用 IOControl 接口暂停输出事件。当所有内容完成后，生产者必须调用 ContentEncoder#complete()。如果不这样做，可能会导致实体被错误地分隔。请注意，ContentEncoder 对象不是线程安全的，应该只在这个方法调用的上下文中使用。当更多内容可用时，IOControl 对象可以共享并在其他线程上使用，从而恢复输出事件通知。

- isRepeatable: Determines whether or not this producer is capable of producing its content more than once. Repeatable content producers are expected to be able to recreate their content even after having been closed.

isRepeatable：确定这个生产者是否能够多次生产其内容。可重复的内容生产者被期望能够重新创造他们的内容，甚至在已经关闭之后。

- close: Closes the producer and releases all resources currently allocated by it.

### 3.7.1. Creating non-blocking entities

Several HTTP entity implementations included in HttpCore NIO support HttpAsyncContentProducer interface:

几个 HTTP 实体实现包括在 HttpCore NIO 支持 HttpAsyncContentProducer 接口：

- [NByteArrayEntity]()

- [NStringEntity]()

- [NFileEntity]()

#### 3.7.1.1. NByteArrayEntity

This is a simple self-contained repeatable entity, which receives its content from a given byte array. This byte array is supplied to the constructor.

这是一个简单的 self-contained 可重复实体，它从给定的字节数组接收其内容。这个字节数组被提供给构造函数。

```
NByteArrayEntity entity = new NByteArrayEntity(new byte[] {1, 2, 3});
```

#### 3.7.1.2. NStringEntity

This is a simple, self-contained, repeatable entity that retrieves its data from a java.lang.String object. It has 2 constructors, one simply constructs with a given string where the other also takes a character encoding for the data in the java.lang.String.

这是一个简单的、self-contained、可重复的实体，它从 java.lang.String 对象中检索数据。它有两个构造函数，一个简单地用给定的字符串构造，另一个也对 java.lang.String 中的数据进行字符编码。

```
NStringEntity myEntity = new NStringEntity("important message",
        Consts.UTF_8);

```

#### 3.7.1.3. NFileEntity

This entity reads its content body from a file. This class is mostly used to stream large files of different types, so one needs to supply the content type of the file to make sure the content can be correctly recognized and processed by the recipient.

此实体从文件中读取其内容主体。此类主要用于传输不同类型的大型文件，因此需要提供文件的内容类型，以确保接收方能够正确识别和处理内容。

```
File staticFile = new File("/path/to/myapp.jar");
NFileEntity entity = new NFileEntity(staticFile,
    ContentType.create("application/java-archive", null));

```

The NHttpEntity will make use of the direct channel I/O whenever possible, provided the content encoder is capable of transferring data directly from a file to the socket of the underlying connection.

只要内容编码器能够将数据直接从文件传输到底层连接的套接字，NHttpEntity 将尽可能使用直接通道 I/O。

## 3.8. Non-blocking HTTP protocol handlers

### 3.8.1. Asynchronous HTTP service

HttpAsyncService is a fully asynchronous HTTP server side protocol handler based on the non-blocking (NIO) I/O model. HttpAsyncService translates individual events fired through the NHttpServerEventHandler interface into logically related HTTP message exchanges.

HttpAsyncService 是一个完全异步的 HTTP 服务器端协议处理程序，它基于非阻塞（NIO）I/O 模型。HttpAsyncService 将通过 NHttpServerEventHandler 接口触发的单个事件转换为逻辑相关的 HTTP 消息交换。

Upon receiving an incoming request the HttpAsyncService verifies the message for compliance with the server expectations using HttpAsyncExpectationVerifier, if provided, and then HttpAsyncRequestHandlerResolver is used to resolve the request URI to a particular HttpAsyncRequestHandler intended to handle the request with the given URI. The protocol handler uses the selected HttpAsyncRequestHandler instance to process the incoming request and to generate an outgoing response.

在接收传入的请求 HttpAsyncService 验证消息服务器为符合预期使用 HttpAsyncExpectationVerifier，如果提供，然后 HttpAsyncRequestHandlerResolver 用于解决特定 HttpAsyncRequestHandler 打算请求 URI 处理请求的 URI。协议处理程序使用所选的 HttpAsyncRequestHandler 实例来处理传入的请求并生成传出的响应。

HttpAsyncService relies on HttpProcessor to generate mandatory protocol headers for all outgoing messages and apply common, cross-cutting message transformations to all incoming and outgoing messages, whereas individual HTTP request handlers are expected to implement application specific content generation and processing.

HttpAsyncService 依赖于 HttpProcessor 来为所有传出消息生成强制的协议头，并将通用的、横切的消息转换应用于所有传入和传出消息，而单个的 HTTP 请求处理程序被期望实现特定于应用程序的内容生成和处理。

```
HttpProcessor httpproc = HttpProcessorBuilder.create()
        .add(new ResponseDate())
        .add(new ResponseServer("MyServer-HTTP/1.1"))
        .add(new ResponseContent())
        .add(new ResponseConnControl())
        .build();
HttpAsyncService protocolHandler = new HttpAsyncService(httpproc, null);
IOEventDispatch ioEventDispatch = new DefaultHttpServerIODispatch(
        protocolHandler,
        new DefaultNHttpServerConnectionFactory(ConnectionConfig.DEFAULT));
ListeningIOReactor ioreactor = new DefaultListeningIOReactor();
ioreactor.execute(ioEventDispatch);
```

#### 3.8.1.1. Non-blocking HTTP request handlers

HttpAsyncRequestHandler represents a routine for asynchronous processing of a specific group of non-blocking HTTP requests. Protocol handlers are designed to take care of protocol specific aspects, whereas individual request handlers are expected to take care of application specific HTTP processing. The main purpose of a request handler is to generate a response object with a content entity to be sent back to the client in response to the given request.

HttpAsyncRequestHandler 表示异步处理一组特定的非阻塞 HTTP 请求的例程。协议处理程序被设计来处理特定于协议的方面，而单个请求处理程序被期望处理特定于应用程序的 HTTP 处理。请求处理程序的主要用途是生成一个响应对象，其中包含一个内容实体，将作为对给定请求的响应发送回客户端。

```
HttpAsyncRequestHandler<HttpRequest> rh = new HttpAsyncRequestHandler<HttpRequest>() {

    public HttpAsyncRequestConsumer<HttpRequest> processRequest(
            final HttpRequest request,
            final HttpContext context) {
        // Buffer request content in memory for simplicity
        return new BasicAsyncRequestConsumer();
    }

    public void handle(
            final HttpRequest request,
            final HttpAsyncExchange httpexchange,
            final HttpContext context) throws HttpException, IOException {
        HttpResponse response = httpexchange.getResponse();
        response.setStatusCode(HttpStatus.SC_OK);
        NFileEntity body = new NFileEntity(new File("static.html"),
                ContentType.create("text/html", Consts.UTF_8));
        response.setEntity(body);
        httpexchange.submitResponse(new BasicAsyncResponseProducer(response));
    }

};
```

Request handlers must be implemented in a thread-safe manner. Similarly to servlets, request handlers should not use instance variables unless access to those variables are synchronized.

请求处理程序必须以线程安全的方式实现。与 servlet 类似，请求处理程序不应该使用实例变量，除非同步了对这些变量的访问。

#### 3.8.1.2. Asynchronous HTTP exchange

The most fundamental difference of the non-blocking request handlers compared to their blocking counterparts is ability to defer transmission of the HTTP response back to the client without blocking the I/O thread by delegating the process of handling the HTTP request to a worker thread or another service. The instance of HttpAsyncExchange passed as a parameter to the HttpAsyncRequestHandler#handle method to submit a response as at a later point once response content becomes available.

与阻塞请求处理程序相比，非阻塞请求处理程序最基本的区别在于，它们能够将 HTTP 响应的传输延迟回客户端，而不需要通过将处理 HTTP 请求的过程委托给工作线程或另一个服务来阻塞 I/O 线程。HttpAsyncExchange 的实例作为参数传递给 HttpAsyncRequestHandler#handle 方法，以便在稍后响应内容可用时提交响应。

The HttpAsyncExchange interface can be interacted with using the following methods:

HttpAsyncExchange 接口可以使用以下方法进行交互：

- getRequest: Returns the received HTTP request message.

getRequest：返回接收到的 HTTP 请求消息。

- getResponse: Returns the default HTTP response message that can submitted once ready.

getResponse：返回默认的 HTTP 响应消息，可以提交一次就绪。

- submitResponse: Submits an HTTP response and completed the message exchange.

submitResponse：提交一个 HTTP 响应并完成消息交换。

- isCompleted: Determines whether or not the message exchange has been completed.

isCompleted：确定消息交换是否已经完成。

- setCallback: Sets Cancellable callback to be invoked in case the underlying connection times out or gets terminated prematurely by the client. This callback can be used to cancel a long running response generating process if a response is no longer needed.

setCallback：设置可取消的回调，以便在底层连接超时或被客户端提前终止的情况下调用。如果不再需要响应，此回调可用于取消长时间运行的响应生成过程。

- setTimeout: Sets timeout for this message exchange.

setTimeout：设置此消息交换的超时。

- getTimeout: Returns timeout for this message exchange.

getTimeout：返回此消息交换的超时。

```
HttpAsyncRequestHandler<HttpRequest> rh = new HttpAsyncRequestHandler<HttpRequest>() {

    public HttpAsyncRequestConsumer<HttpRequest> processRequest(
            final HttpRequest request,
            final HttpContext context) {
        // Buffer request content in memory for simplicity
        return new BasicAsyncRequestConsumer();
    }

    public void handle(
            final HttpRequest request,
            final HttpAsyncExchange httpexchange,
            final HttpContext context) throws HttpException, IOException {

        new Thread() {

            @Override
            public void run() {
                try {
                    Thread.sleep(10);
                }
                catch(InterruptedException ie) {}
                HttpResponse response = httpexchange.getResponse();
                response.setStatusCode(HttpStatus.SC_OK);
                NFileEntity body = new NFileEntity(new File("static.html"),
                        ContentType.create("text/html", Consts.UTF_8));
                response.setEntity(body);
                httpexchange.submitResponse(new BasicAsyncResponseProducer(response));
            }
        }.start();

    }

};
```

Please note HttpResponse instances are not thread-safe and may not be modified concurrently. Non-blocking request handlers must ensure HTTP response cannot be accessed by more than one thread at a time.

请注意，HttpResponse 实例不是线程安全的，不能同时修改。非阻塞请求处理程序必须确保一次不能有多个线程访问 HTTP 响应。

#### 3.8.1.3. Asynchronous HTTP request consumer

HttpAsyncRequestConsumer facilitates the process of asynchronous processing of HTTP requests. It is a callback interface used by HttpAsyncRequestHandlers to process an incoming HTTP request message and to stream its content from a non-blocking server side HTTP connection.

HttpAsyncRequestConsumer 简化了 HTTP 请求的异步处理过程。它是一个回调接口，HttpAsyncRequestHandlers 使用它来处理传入的 HTTP 请求消息，并从非阻塞的服务器端 HTTP 连接传输其内容。

HTTP I/O events and methods as defined by the HttpAsyncRequestConsumer interface:

HttpAsyncRequestConsumer 接口定义的 HTTP I/O 事件和方法：

- requestReceived: Invoked when a HTTP request message is received.

requestReceived：在接收到 HTTP 请求消息时调用。

- consumeContent: Invoked to process a chunk of content from the ContentDecoder. The IOControl interface can be used to suspend input events if the consumer is temporarily unable to consume more content. The consumer can use the ContentDecoder#isCompleted() method to find out whether or not the message content has been fully consumed. Please note that the ContentDecoder object is not thread-safe and should only be used within the context of this method call. The IOControl object can be shared and used on other thread to resume input event notifications when the consumer is capable of processing more content. This event is invoked only if the incoming request message has a content entity enclosed in it.

consumeContent：用于处理 ContentDecoder 中的内容块。如果使用者暂时无法消费更多内容，可以使用 IOControl 接口来暂停输入事件。使用者可以使用 ContentDecoder#isCompleted()方法来确定消息内容是否已被完全使用。请注意，ContentDecoder 对象不是线程安全的，应该只在这个方法调用的上下文中使用。当使用者能够处理更多内容时，可以在其他线程上共享和使用 IOControl 对象来恢复输入事件通知。只有在传入请求消息中包含内容实体时才调用此事件。

- requestCompleted: Invoked to signal that the request has been fully processed.

requestCompleted：请求已完全处理。

- failed: Invoked to signal that the request processing terminated abnormally.

failed：请求处理异常终止。

- getException: Returns an exception in case of an abnormal termination. This method returns null if the request execution is still ongoing or if it completed successfully.

getException：返回非正常终止情况下的异常。如果请求执行仍在进行中或成功完成，则此方法返回 null。

- getResult: Returns a result of the request execution, when available. This method returns null if the request execution is still ongoing.

getResult：返回请求执行的结果（如果可用）。如果请求执行仍在进行，则此方法返回 null。

- isDone: Determines whether or not the request execution completed. If the request processing terminated normally getResult() can be used to obtain the result. If the request processing terminated abnormally getException() can be used to obtain the cause.

isDone：确定请求执行是否完成。如果请求处理正常终止，可以使用 getResult() 来获得结果。如果请求处理异常终止，可以使用 getException() 获取原因。

- close: Closes the consumer and releases all resources currently allocated by it.

close：关闭消费者并释放所有当前分配给它的资源。

HttpAsyncRequestConsumer implementations are expected to be thread-safe.

HttpAsyncRequestConsumer 实现应该是线程安全的。

BasicAsyncRequestConsumer is a very basic implementation of the HttpAsyncRequestConsumer interface shipped with the library. Please note that this consumer buffers request content in memory and therefore should be used for relatively small request messages.

BasicAsyncRequestConsumer 是该库附带的 HttpAsyncRequestConsumer 接口的一个非常基本的实现。请注意，这个使用者缓冲内存中的请求内容，因此应该用于相对较小的请求消息。

#### 3.8.1.4. Asynchronous HTTP response producer

HttpAsyncResponseProducer facilitates the process of asynchronous generation of HTTP responses. It is a callback interface used by HttpAsyncRequestHandlers to generate an HTTP response message and to stream its content to a non-blocking server side HTTP connection.

HttpAsyncResponseProducer 简化了异步生成 HTTP 响应的过程。它是 HttpAsyncRequestHandlers 使用的回调接口，用于生成 HTTP 响应消息并将其内容流到非阻塞的服务器端 HTTP 连接。

HTTP I/O events and methods as defined by the HttpAsyncResponseProducer interface:

HttpAsyncResponseProducer 接口定义的 HTTP I/O 事件和方法:

- generateResponse: Invoked to generate a HTTP response message header.

generateResponse：被调用以生成 HTTP 响应消息头。

- produceContent: Invoked to write out a chunk of content to the ContentEncoder. The IOControl interface can be used to suspend output events if the producer is temporarily unable to produce more content. When all content is finished, the producer MUST call ContentEncoder#complete(). Failure to do so may cause the entity to be incorrectly delimited. Please note that the ContentEncoder object is not thread-safe and should only be used within the context of this method call. The IOControl object can be shared and used on other thread resume output event notifications when more content is made available. This event is invoked only for if the outgoing response message has a content entity enclosed in it, that is HttpResponse#getEntity() returns null.

- responseCompleted: Invoked to signal that the response has been fully written out.

responseCompleted：被调用来表示响应已经完全写出来。

- failed: Invoked to signal that the response processing terminated abnormally.

- close: Closes the producer and releases all resources currently allocated by it.

HttpAsyncResponseProducer implementations are expected to be thread-safe.

BasicAsyncResponseProducer is a basic implementation of the HttpAsyncResponseProducer interface shipped with the library. The producer can make use of the HttpAsyncContentProducer interface to efficiently stream out message content to a non-blocking HTTP connection, if it is implemented by the HttpEntity enclosed in the response.

#### 3.8.1.5. Non-blocking request handler resolver

The management of non-blocking HTTP request handlers is quite similar to that of blocking HTTP request handlers. Usually an instance of HttpAsyncRequestHandlerResolver is used to maintain a registry of request handlers and to matches a request URI to a particular request handler. HttpCore includes only a very simple implementation of the request handler resolver based on a trivial pattern matching algorithm: HttpAsyncRequestHandlerRegistry supports only three formats: `*`, `<uri>*` and `*<uri>`.

非阻塞 HTTP 请求处理程序的管理与阻塞 HTTP 请求处理程序的管理非常相似。HttpAsyncRequestHandlerResolver 的实例通常用于维护请求处理程序的注册表，并将请求 URI 与特定的请求处理程序相匹配。HttpCore 只包含了一个非常简单的基于普通模式匹配算法的请求处理解析器实现：HttpAsyncRequestHandlerRegistry 只支持三种格式：`*`、`<uri>*` 和 `*<uri>`

```
HttpAsyncRequestHandler<?> myRequestHandler1 = <...>
HttpAsyncRequestHandler<?> myRequestHandler2 = <...>
HttpAsyncRequestHandler<?> myRequestHandler3 = <...>
UriHttpAsyncRequestHandlerMapper handlerReqistry =
        new UriHttpAsyncRequestHandlerMapper();
handlerReqistry.register("/service/*", myRequestHandler1);
handlerReqistry.register("*.do", myRequestHandler2);
handlerReqistry.register("*", myRequestHandler3);
```

Users are encouraged to provide more sophisticated implementations of HttpAsyncRequestHandlerResolver, for instance, based on regular expressions.

例如，我们鼓励用户基于正则表达式提供更复杂的 HttpAsyncRequestHandlerResolver 实现。

### 3.8.2. Asynchronous HTTP request executor

HttpAsyncRequestExecutor is a fully asynchronous client side HTTP protocol handler based on the NIO (non-blocking) I/O model. HttpAsyncRequestExecutor translates individual events fired through the NHttpClientEventHandler interface into logically related HTTP message exchanges.

HttpAsyncRequestExecutor 是一个基于 NIO（非阻塞）I/O 模型的完全异步的客户端 HTTP 协议处理程序。HttpAsyncRequestExecutor 将通过 NHttpClientEventHandler 接口触发的单个事件转换为逻辑相关的 HTTP 消息交换。

HttpAsyncRequestExecutor relies on HttpAsyncRequestExecutionHandler to implement application specific content generation and processing and to handle logically related series of HTTP request / response exchanges, which may also span across multiple connections. HttpProcessor provided by the HttpAsyncRequestExecutionHandler instance will be used to generate mandatory protocol headers for all outgoing messages and apply common, cross-cutting message transformations to all incoming and outgoing messages. The caller is expected to pass an instance of HttpAsyncRequestExecutionHandler to be used for the next series of HTTP message exchanges through the connection context using HttpAsyncRequestExecutor#HTTP_HANDLER attribute. HTTP exchange sequence is considered complete when the HttpAsyncRequestExecutionHandler#isDone() method returns true.

HttpAsyncRequestExecutor 依赖 HttpAsyncRequestExecutionHandler 实现特定于应用程序的内容生成和处理，并处理逻辑相关的一系列 HTTP 请求/响应交换，这些交换也可能跨多个连接。HttpAsyncRequestExecutionHandler 实例提供的 HttpProcessor 将用于为所有传出消息生成强制协议头，并将通用的 cross-cutting 消息转换应用于所有传入和传出消息。调用者希望通过使用 HttpAsyncRequestExecutor#HTTP_HANDLER 属性的连接上下文传递一个 HttpAsyncRequestExecutionHandler 实例，用于下一系列 HTTP 消息交换。当 HttpAsyncRequestExecutionHandler#isDone() 方法返回 true 时，HTTP 交换序列被认为是完整的。

```
HttpAsyncRequestExecutor ph = new HttpAsyncRequestExecutor();
IOEventDispatch ioEventDispatch = new DefaultHttpClientIODispatch(ph,
        new DefaultNHttpClientConnectionFactory(ConnectionConfig.DEFAULT));
ConnectingIOReactor ioreactor = new DefaultConnectingIOReactor();
ioreactor.execute(ioEventDispatch);
```

The HttpAsyncRequester utility class can be used to abstract away low level details of HttpAsyncRequestExecutionHandler management. Please note HttpAsyncRequester supports single HTTP request / response exchanges only. It does not support HTTP authentication and does not handle redirects automatically.

HttpAsyncRequester 实用程序类可用于抽象出 HttpAsyncRequestExecutionHandler 管理的低层细节。请注意，HttpAsyncRequester 只支持单个 HTTP 请求/响应交换。它不支持 HTTP 身份验证，也不自动处理重定向。

```
HttpProcessor httpproc = HttpProcessorBuilder.create()
        .add(new RequestContent())
        .add(new RequestTargetHost())
        .add(new RequestConnControl())
        .add(new RequestUserAgent("MyAgent-HTTP/1.1"))
        .add(new RequestExpectContinue(true))
        .build();
HttpAsyncRequester requester = new HttpAsyncRequester(httpproc);
NHttpClientConnection conn = <...>
Future<HttpResponse> future = requester.execute(
        new BasicAsyncRequestProducer(
                new HttpHost("localhost"),
                new BasicHttpRequest("GET", "/")),
        new BasicAsyncResponseConsumer(),
        conn);
HttpResponse response = future.get();
```

#### 3.8.2.1. Asynchronous HTTP request producer

HttpAsyncRequestProducer facilitates the process of asynchronous generation of HTTP requests. It is a callback interface whose methods get invoked to generate an HTTP request message and to stream message content to a non-blocking client side HTTP connection.

Repeatable request producers capable of generating the same request message more than once can be reset to their initial state by calling the resetRequest() method, at which point request producers are expected to release currently allocated resources that are no longer needed or re-acquire resources needed to repeat the process.

HTTP I/O events and methods as defined by the HttpAsyncRequestProducer interface:

- getTarget: Invoked to obtain the request target host.

- generateRequest: Invoked to generate a HTTP request message header. The message is expected to implement the HttpEntityEnclosingRequest interface if it is to enclose a content entity.

- produceContent: Invoked to write out a chunk of content to the ContentEncoder. The IOControl interface can be used to suspend output events if the producer is temporarily unable to produce more content. When all content is finished, the producer MUST call ContentEncoder#complete(). Failure to do so may cause the entity to be incorrectly delimited. Please note that the ContentEncoder object is not thread-safe and should only be used within the context of this method call. The IOControl object can be shared and used on other thread resume output event notifications when more content is made available. This event is invoked only for if the outgoing request message has a content entity enclosed in it, that is HttpEntityEnclosingRequest#getEntity() returns null .

- requestCompleted: Invoked to signal that the request has been fully written out.

- failed: Invoked to signal that the request processing terminated abnormally.

- resetRequest: Invoked to reset the producer to its initial state. Repeatable request producers are expected to release currently allocated resources that are no longer needed or re-acquire resources needed to repeat the process.

- close: Closes the producer and releases all resources currently allocated by it.

HttpAsyncRequestProducer implementations are expected to be thread-safe.

HttpAsyncRequestProducer 实现应该是线程安全的。

BasicAsyncRequestProducer is a basic implementation of the HttpAsyncRequestProducer interface shipped with the library. The producer can make use of the HttpAsyncContentProducer interface to efficiently stream out message content to a non-blocking HTTP connection, if it is implemented by the HttpEntity enclosed in the request.

#### 3.8.2.2. Asynchronous HTTP response consumer

HttpAsyncResponseConsumer facilitates the process of asynchronous processing of HTTP responses. It is a callback interface whose methods get invoked to process an HTTP response message and to stream message content from a non-blocking client side HTTP connection.

HTTP I/O events and methods as defined by the HttpAsyncResponseConsumer interface:

- responseReceived: Invoked when a HTTP response message is received.

- consumeContent: Invoked to process a chunk of content from the ContentDecoder. The IOControl interface can be used to suspend input events if the consumer is temporarily unable to consume more content. The consumer can use the ContentDecoder#isCompleted() method to find out whether or not the message content has been fully consumed. Please note that the ContentDecoder object is not thread-safe and should only be used within the context of this method call. The IOControl object can be shared and used on other thread to resume input event notifications when the consumer is capable of processing more content. This event is invoked only for if the incoming response message has a content entity enclosed in it.

- responseCompleted: Invoked to signal that the response has been fully processed.

- failed: Invoked to signal that the response processing terminated abnormally.

- getException: Returns an exception in case of an abnormal termination. This method returns null if the response processing is still ongoing or if it completed successfully.

- getResult: Returns a result of the response processing, when available. This method returns null if the response processing is still ongoing.

- isDone: Determines whether or not the response processing completed. If the response processing terminated normally getResult() can be used to obtain the result. If the response processing terminated abnormally getException() can be used to obtain the cause.

- close: Closes the consumer and releases all resources currently allocated by it.

HttpAsyncResponseConsumer implementations are expected to be thread-safe.

BasicAsyncResponseConsumer is a very basic implementation of the HttpAsyncResponseConsumer interface shipped with the library. Please note that this consumer buffers response content in memory and therefore should be used for relatively small response messages.

BasicAsyncResponseConsumer 是该库附带的 HttpAsyncResponseConsumer 接口的一个非常基本的实现。请注意，此使用者缓冲内存中的响应内容，因此应该用于相对较小的响应消息。

## 3.9. Non-blocking connection pools

Non-blocking connection pools are quite similar to blocking one with one significant distinction that they have to reply an I/O reactor to establish new connections. As a result connections leased from a non-blocking pool are returned fully initialized and already bound to a particular I/O session. Non-blocking connections managed by a connection pool cannot be bound to an arbitrary I/O session.

非阻塞连接池与阻塞连接池非常相似，但有一个重要区别：它们必须响应 I/O reactor 来建立新连接。结果，从非阻塞池租用的连接被完全初始化并已绑定到特定的 I/O 会话。不能将由连接池管理的非阻塞连接绑定到任意 I/O 会话。

```
HttpHost target = new HttpHost("localhost");
ConnectingIOReactor ioreactor = <...>
BasicNIOConnPool connpool = new BasicNIOConnPool(ioreactor);
connpool.lease(target, null,
        10, TimeUnit.SECONDS,
        new FutureCallback<BasicNIOPoolEntry>() {
            @Override
            public void completed(BasicNIOPoolEntry entry) {
                NHttpClientConnection conn = entry.getConnection();
                System.out.println("Connection successfully leased");
                // Update connection context and request output
                conn.requestOutput();
            }

            @Override
            public void failed(Exception ex) {
                System.out.println("Connection request failed");
                ex.printStackTrace();
            }

            @Override
            public void cancelled() {
            }
        });
```

Please note due to event-driven nature of asynchronous communication model it is quite difficult to ensure proper release of persistent connections back to the pool. One can make use of HttpAsyncRequester to handle connection lease and release behind the scene.

请注意，由于异步通信模型的事件驱动特性，很难确保将持久连接释放回池。可以使用 HttpAsyncRequester 在后台处理连接的租用和释放。

```
ConnectingIOReactor ioreactor = <...>
HttpProcessor httpproc = <...>
BasicNIOConnPool connpool = new BasicNIOConnPool(ioreactor);
HttpAsyncRequester requester = new HttpAsyncRequester(httpproc);
HttpHost target = new HttpHost("localhost");
Future<HttpResponse> future = requester.execute(
        new BasicAsyncRequestProducer(
                new HttpHost("localhost"),
                new BasicHttpRequest("GET", "/")),
        new BasicAsyncResponseConsumer(),
        connpool);
```

## 3.10. Pipelined request execution

In addition to the normal request / response execution mode HttpAsyncRequester is also capable of executing requests in the so called pipelined mode whereby several requests are immediately written out to the underlying connection. Please note that entity enclosing requests can be executed in the pipelined mode but the 'expect: continue' handshake should be disabled (request messages should contains no 'Expect: 100-continue' header).

除了正常的请求/响应执行模式外，HttpAsyncRequester 还能够在所谓的流水线模式中执行请求，在流水线模式中，几个请求被立即写入底层连接。请注意，包含请求的实体可以在流水线模式下执行，但是应该禁用 'expect: continue' 握手（请求消息应该不包含 'Expect: 100-continue' 头）。

```
HttpProcessor httpproc = <...>
HttpAsyncRequester requester = new HttpAsyncRequester(httpproc);
HttpHost target = new HttpHost("www.apache.org");
List<BasicAsyncRequestProducer> requestProducers = Arrays.asList(
        new BasicAsyncRequestProducer(target, new BasicHttpRequest("GET", "/index.html")),
        new BasicAsyncRequestProducer(target, new BasicHttpRequest("GET", "/foundation/index.html")),
        new BasicAsyncRequestProducer(target, new BasicHttpRequest("GET", "/foundation/how-it-works.html"))
);
List<BasicAsyncResponseConsumer> responseConsumers = Arrays.asList(
        new BasicAsyncResponseConsumer(),
        new BasicAsyncResponseConsumer(),
        new BasicAsyncResponseConsumer()
);
HttpCoreContext context = HttpCoreContext.create();
Future<List<HttpResponse>> future = requester.executePipelined(
        target, requestProducers, responseConsumers, pool, context, null);
```

Please note that older web servers and especially older HTTP proxies may be unable to handle pipelined requests correctly. Use the pipelined execution mode with caution.

请注意，较旧的 web 服务器，尤其是较旧的 HTTP 代理可能无法正确处理流水线请求。小心使用流水线执行模式。

## 3.11. Non-blocking TLS/SSL

### 3.11.1. SSL I/O session

SSLIOSession is a decorator class intended to transparently extend any arbitrary IOSession with transport layer security capabilities based on the SSL/TLS protocol. Default HTTP connection implementations and protocol handlers should be able to work with SSL sessions without special preconditions or modifications.

SSLIOSession 是一个装饰类，用于透明地扩展任何具有基于 SSL/TLS 协议的传输层安全功能的 IOSession。默认的 HTTP 连接实现和协议处理程序应该能够在没有特殊先决条件或修改的情况下使用 SSL 会话。

```
SSLContext sslcontext = SSLContext.getInstance("Default");
sslcontext.init(null, null, null);
// Plain I/O session
IOSession iosession = <...>
SSLIOSession sslsession = new SSLIOSession(
        iosession, SSLMode.CLIENT, sslcontext, null);
iosession.setAttribute(SSLIOSession.SESSION_KEY, sslsession);
NHttpClientConnection conn = new DefaultNHttpClientConnection(
        sslsession, 8 * 1024);
```

One can also use SSLNHttpClientConnectionFactory or SSLNHttpServerConnectionFactory classes to conveniently create SSL encrypterd HTTP connections.

您还可以使用 SSLNHttpClientConnectionFactory 或 SSLNHttpServerConnectionFactory 类来方便地创建 SSL encrypterd HTTP 连接。

```
SSLContext sslcontext = SSLContext.getInstance("Default");
sslcontext.init(null, null, null);
// Plain I/O session
IOSession iosession = <...>
SSLNHttpClientConnectionFactory connfactory = new SSLNHttpClientConnectionFactory(
        sslcontext, null, ConnectionConfig.DEFAULT);
NHttpClientConnection conn = connfactory.createConnection(iosession);
```

#### 3.11.1.1. SSL setup handler

Applications can customize various aspects of the TLS/SSl protocol by passing a custom implementation of the SSLSetupHandler interface.

应用程序可以通过传递 SSLSetupHandler 接口的自定义实现来定制 TLS/SSl 协议的各个方面。

SSL events as defined by the SSLSetupHandler interface:

SSLSetupHandler 接口定义的 SSL 事件：

- initalize: Triggered when the SSL connection is being initialized. The handler can use this callback to customize properties of the javax.net.ssl.SSLEngine used to establish the SSL session.

initalize：初始化 SSL 连接时触发。处理程序可以使用这个回调来定制用于建立 SSL 会话的 javax.net.ssl.SSLEngine 的属性。

- verify: Triggered when the SSL connection has been established and initial SSL handshake has been successfully completed. The handler can use this callback to verify properties of the SSLSession. For instance this would be the right place to enforce SSL cipher strength, validate certificate chain and do hostname checks.

当 SSL 连接已建立且初始 SSL 握手已成功完成时触发。处理程序可以使用此回调来验证 SSLSession 的属性。例如，这是执行 SSL 加密强度、验证证书链和执行主机名检查的正确位置。

```
SSLContext sslcontext = SSLContexts.createDefault();
// Plain I/O session
IOSession iosession = <...>

SSLIOSession sslsession = new SSLIOSession(
        iosession, SSLMode.CLIENT, sslcontext, new SSLSetupHandler() {

    public void initalize(final SSLEngine sslengine) throws SSLException {
        // Enforce TLS and disable SSL
        sslengine.setEnabledProtocols(new String[] {
                "TLSv1",
                "TLSv1.1",
                "TLSv1.2" });
        // Enforce strong ciphers
        sslengine.setEnabledCipherSuites(new String[] {
                "TLS_RSA_WITH_AES_256_CBC_SHA",
                "TLS_DHE_RSA_WITH_AES_256_CBC_SHA",
                "TLS_DHE_DSS_WITH_AES_256_CBC_SHA" });
    }

    public void verify(
            final IOSession iosession,
            final SSLSession sslsession) throws SSLException {
        X509Certificate[] certs = sslsession.getPeerCertificateChain();
        // Examine peer certificate chain
        for (X509Certificate cert: certs) {
            System.out.println(cert.toString());
        }
    }

});
```

SSLSetupHandler impelemntations can also be used with the SSLNHttpClientConnectionFactory or SSLNHttpServerConnectionFactory classes.

**译注：原文有拼写错误，正确为 implementations**

SSLSetupHandler 实现还可以与 SSLNHttpClientConnectionFactory 或 SSLNHttpServerConnectionFactory 类一起使用。

```
SSLContext sslcontext = SSLContexts.createDefault();
// Plain I/O session
IOSession iosession = <...>

SSLSetupHandler mysslhandler = new SSLSetupHandler() {

    public void initalize(final SSLEngine sslengine) throws SSLException {
        // Enforce TLS and disable SSL
        sslengine.setEnabledProtocols(new String[] {
                "TLSv1",
                "TLSv1.1",
                "TLSv1.2" });
    }

    public void verify(
            final IOSession iosession, final SSLSession sslsession) throws SSLException {
    }


};
SSLNHttpClientConnectionFactory connfactory = new SSLNHttpClientConnectionFactory(
        sslcontext, mysslhandler, ConnectionConfig.DEFAULT);
NHttpClientConnection conn = connfactory.createConnection(iosession);
```

### 3.11.2. TLS/SSL aware I/O event dispatches

Default IOEventDispatch implementations shipped with the library such as DefaultHttpServerIODispatch and DefaultHttpClientIODispatch automatically detect SSL encrypted sessions and handle SSL transport aspects transparently. However, custom I/O event dispatchers that do not extend AbstractIODispatch are required to take some additional actions to ensure correct functioning of the transport layer encryption.

默认的 IOEventDispatch 实现随库一起提供，如 DefaultHttpServerIODispatch 和 DefaultHttpClientIODispatch 自动检测 SSL 加密会话并透明地处理 SSL 传输方面。但是，不扩展 AbstractIODispatch 的自定义 I/O 事件分发程序需要采取一些额外的操作来确保传输层加密的正确运行。

- The I/O dispatch may need to call SSLIOSession#initalize() method in order to put the SSL session either into a client or a server mode, if the SSL session has not been yet initialized.

I/O 分派可能需要调用 SSLIOSession#initalize() 方法，以便在 SSL 会话尚未初始化的情况下将 SSL 会话放入客户机或服务器模式。

- When the underlying I/O session is input ready, the I/O dispatcher should check whether the SSL I/O session is ready to produce input data by calling SSLIOSession#isAppInputReady(), pass control to the protocol handler if it is, and finally call SSLIOSession#inboundTransport() method in order to do the necessary SSL handshaking and decrypt input data.

当底层 I/O 会话准备好输入时，I/O dispatcher 应该通过调用 SSLIOSession#isAppInputReady() 来检查 SSL I/O 会话是否准备好生成输入数据，如果准备好了，则将控制权传递给协议处理程序，最后调用 SSLIOSession#inboundTransport() 方法，以便进行必要的 SSL 握手并解密输入数据。

- When the underlying I/O session is output ready, the I/O dispatcher should check whether the SSL I/O session is ready to accept output data by calling SSLIOSession#isAppOutputReady(), pass control to the protocol handler if it is, and finally call SSLIOSession#outboundTransport() method in order to do the necessary SSL handshaking and encrypt application data.

当底层 I/O 会话准备好输出时，I/O dispatcher 应该通过调用 SSLIOSession#isAppOutputReady() 来检查 SSL I/O 会话是否准备好接收输出数据，如果准备好了，则将控制权传递给协议处理程序，最后调用 SSLIOSession#outboundTransport() 方法，以便进行必要的 SSL 握手并加密应用数据。

## 3.12. Embedded non-blocking HTTP server

As of version 4.4 HttpCore ships with an embedded non-blocking HTTP server based on non-blocking I/O components described above.

从版本 4.4 开始，HttpCore 附带了一个基于上述非阻塞 I/O 组件的嵌入式非阻塞 HTTP 服务器。

```
HttpAsyncRequestHandler<?> requestHandler = <...>
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
