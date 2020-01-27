# Chapter 3. Asynchronous I/O based on NIO

Asynchronous I/O model may be more appropriate for those scenarios where raw data throughput is less important than the ability to handle thousands of simultaneous connections in a scalable, resource efficient manner. Asynchronous I/O is arguably more complex and usually requires a special care when dealing with large message payloads.

## 3.1. Differences from other I/O frameworks

Solves similar problems as other frameworks, but has certain distinct features:

- minimalistic, optimized for data volume intensive protocols such as HTTP.

- efficient memory management: data consumer can read is only as much input data as it can process without having to allocate more memory.

- direct access to the NIO channels where possible.

## 3.2. I/O reactor

HttpCore NIO is based on the Reactor pattern as described by Doug Lea. The purpose of I/O reactors is to react to I/O events and to dispatch event notifications to individual I/O sessions. The main idea of I/O reactor pattern is to break away from the one thread per connection model imposed by the classic blocking I/O model. The IOReactor interface represents an abstract object which implements the Reactor pattern. Internally, IOReactor implementations encapsulate functionality of the NIO java.nio.channels.Selector.

I/O reactors usually employ a small number of dispatch threads (often as few as one) to dispatch I/O event notifications to a much greater number (often as many as several thousands) of I/O sessions or connections. It is generally recommended to have one dispatch thread per CPU core.

```
IOReactorConfig config = IOReactorConfig.DEFAULT;
IOReactor ioreactor = new DefaultConnectingIOReactor(config);
```

### 3.2.1. I/O dispatchers

IOReactor implementations make use of the IOEventDispatch interface to notify clients of events pending for a particular session. All methods of the IOEventDispatch are executed on a dispatch thread of the I/O reactor. Therefore, it is important that processing that takes place in the event methods will not block the dispatch thread for too long, as the I/O reactor will be unable to react to other events.

```
IOReactor ioreactor = new DefaultConnectingIOReactor();

IOEventDispatch eventDispatch = <...>
ioreactor.execute(eventDispatch);
```

Generic I/O events as defined by the IOEventDispatch interface:

- connected: Triggered when a new session has been created.

- inputReady: Triggered when the session has pending input.

- outputReady: Triggered when the session is ready for output.

- timeout: Triggered when the session has timed out.

- disconnected: Triggered when the session has been terminated.

### 3.2.2. I/O reactor shutdown

The shutdown of I/O reactors is a complex process and may usually take a while to complete. I/O reactors will attempt to gracefully terminate all active I/O sessions and dispatch threads approximately within the specified grace period. If any of the I/O sessions fails to terminate correctly, the I/O reactor will forcibly shut down remaining sessions.

```
IOReactor ioreactor = <...>
long gracePeriod = 3000L; // milliseconds
ioreactor.shutdown(gracePeriod);
```

The IOReactor#shutdown(long) method is safe to call from any thread.

### 3.2.3. I/O sessions

The IOSession interface represents a sequence of logically related data exchanges between two end points. IOSession encapsulates functionality of NIO java.nio.channels.SelectionKey and java.nio.channels.SocketChannel. The channel associated with the IOSession can be used to read data from and write data to the session.

```
IOSession iosession = <...>
ReadableByteChannel ch = (ReadableByteChannel) iosession.channel();
ByteBuffer dst = ByteBuffer.allocate(2048);
ch.read(dst);
```

### 3.2.4. I/O session state management

I/O sessions are not bound to an execution thread, therefore one cannot use the context of the thread to store a session's state. All details about a particular session must be stored within the session itself.

```
IOSession iosession = <...>
Object someState = <...>
iosession.setAttribute("state", someState);
...
IOSession iosession = <...>
Object currentState = iosession.getAttribute("state");
```

Please note that if several sessions make use of shared objects, access to those objects must be made thread-safe.

### 3.2.5. I/O session event mask

One can declare an interest in a particular type of I/O events for a particular I/O session by setting its event mask.

```
IOSession iosession = <...>
iosession.setEventMask(SelectionKey.OP_READ | SelectionKey.OP_WRITE);
```

One can also toggle OP_READ and OP_WRITE flags individually.

```
IOSession iosession = <...>
iosession.setEvent(SelectionKey.OP_READ);
iosession.clearEvent(SelectionKey.OP_READ);
```

Event notifications will not take place if the corresponding interest flag is not set.

### 3.2.6. I/O session buffers

Quite often I/O sessions need to maintain internal I/O buffers in order to transform input / output data prior to returning it to the consumer or writing it to the underlying channel. Memory management in HttpCore NIO is based on the fundamental principle that the data a consumer can read, is only as much input data as it can process without having to allocate more memory. That means, quite often some input data may remain unread in one of the internal or external session buffers. The I/O reactor can query the status of these session buffers, and make sure the consumer gets notified correctly as more data gets stored in one of the session buffers, thus allowing the consumer to read the remaining data once it is able to process it. I/O sessions can be made aware of the status of external session buffers using the SessionBufferStatus interface.

```
IOSession iosession = <...>
SessionBufferStatus myBufferStatus = <...>
iosession.setBufferStatus(myBufferStatus);
iosession.hasBufferedInput();
iosession.hasBufferedOutput();
```

### 3.2.7. I/O session shutdown

One can close an I/O session gracefully by calling IOSession#close() allowing the session to be closed in an orderly manner or by calling IOSession#shutdown() to forcibly close the underlying channel. The distinction between two methods is of primary importance for those types of I/O sessions that involve some sort of a session termination handshake such as SSL/TLS connections.

### 3.2.8. Listening I/O reactors

ListeningIOReactor represents an I/O reactor capable of listening for incoming connections on one or several ports.

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

One can obtain a set of registered endpoints at runtime, query the status of an endpoint at runtime, and close it if desired.

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

```
ConnectingIOReactor ioreactor = <...>

SessionRequest sessionRequest = ioreactor.connect(
        new InetSocketAddress("www.google.com", 80),
        null, null, null);
```

Opening a connection to a remote host usually tends to be a time consuming process and may take a while to complete. One can monitor and control the process of session initialization by means of the SessionRequestinterface.

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

```
if (!sessionRequest.isCompleted()) {
    sessionRequest.cancel();
}
```

One can pass several optional parameters to the ConnectingIOReactor#connect() method to exert a greater control over the process of session initialization.

A non-null local socket address parameter can be used to bind the socket to a specific local address.

```
ConnectingIOReactor ioreactor = <...>

SessionRequest sessionRequest = ioreactor.connect(
        new InetSocketAddress("www.google.com", 80),
        new InetSocketAddress("192.168.0.10", 1234),
        null, null);
```

One can provide an attachment object, which will be added to the new session's context upon initialization. This object can be used to pass an initial processing state to the protocol handler.

```
SessionRequest sessionRequest = ioreactor.connect(
        new InetSocketAddress("www.google.com", 80),
        null, new HttpHost("www.google.ru"), null);

IOSession iosession = sessionRequest.getSession();
HttpHost virtualHost = (HttpHost) iosession.getAttribute(
    IOSession.ATTACHMENT_KEY);
```

It is often desirable to be able to react to the completion of a session request asynchronously without having to wait for it, blocking the current thread of execution. One can optionally provide an implementation SessionRequestCallback interface to get notified of events related to session requests, such as request completion, cancellation, failure or timeout.

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

```
IOReactorConfig config = IOReactorConfig.custom()
        .setInterestOpQueued(true)
        .build();
```

## 3.4. I/O reactor exception handling

Protocol specific exceptions as well as those I/O exceptions thrown in the course of interaction with the session's channel are to be expected and are to be dealt with by specific protocol handlers. These exceptions may result in termination of an individual session but should not affect the I/O reactor and all other active sessions. There are situations, however, when the I/O reactor itself encounters an internal problem such as an I/O exception in the underlying NIO classes or an unhandled runtime exception. Those types of exceptions are usually fatal and will cause the I/O reactor to shut down automatically.

There is a possibility to override this behavior and prevent I/O reactors from shutting down automatically in case of a runtime exception or an I/O exception in internal classes. This can be accomplished by providing a custom implementation of the IOReactorExceptionHandler interface.

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

### 3.4.1. I/O reactor audit log

If an I/O reactor is unable to automatically recover from an I/O or a runtime exception it will enter the shutdown mode. First off, it will close all active listeners and cancel all pending new session requests. Then it will attempt to close all active I/O sessions gracefully giving them some time to flush pending output data and terminate cleanly. Lastly, it will forcibly shut down those I/O sessions that still remain active after the grace period. This is a fairly complex process, where many things can fail at the same time and many different exceptions can be thrown in the course of the shutdown process. The I/O reactor will record all exceptions thrown during the shutdown process, including the original one that actually caused the shutdown in the first place, in an audit log. One can examine the audit log and decide whether it is safe to restart the I/O reactor.

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

### 3.5.1. Execution context of non-blocking HTTP connections

Non-blocking HTTP connections are not bound to a particular thread of execution and therefore they need to maintain their own execution context. Each non-blocking HTTP connection has an HttpContext instance associated with it, which can be used to maintain a processing state. The HttpContext instance is thread-safe and can be manipulated from multiple threads.

```
DefaultNHttpClientConnection conn = <...>
Object myStateObject = <...>

HttpContext context = conn.getContext();
context.setAttribute("state", myStateObject);
```

### 3.5.2. Working with non-blocking HTTP connections

At any point of time one can obtain the request and response objects currently being transferred over the non-blocking HTTP connection. Any of these objects, or both, can be null if there is no incoming or outgoing message currently being transferred.

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

Over-simplified process of submitting a request on the client side may look like this:

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

### 3.5.3. HTTP I/O control

All non-blocking HTTP connections classes implement IOControl interface, which represents a subset of connection functionality for controlling interest in I/O even notifications. IOControl instances are expected to be fully thread-safe. Therefore IOControl can be used to request / suspend I/O event notifications from any thread.

One must take special precautions when interacting with non-blocking connections. HttpRequest and HttpResponse are not thread-safe. It is generally advisable that all input / output operations on a non-blocking connection are executed from the I/O event dispatch thread.

The following pattern is recommended:

- Use IOControl interface to pass control over connection's I/O events to another thread / session.

- If input / output operations need be executed on that particular connection, store all the required information (state) in the connection context and request the appropriate I/O operation by calling IOControl#requestInput() or IOControl#requestOutput() method.

- Execute the required operations from the event method on the dispatch thread using information stored in connection context.

Please note all operations that take place in the event methods should not block for too long, because while the dispatch thread remains blocked in one session, it is unable to process events for all other sessions. I/O operations with the underlying channel of the session are not a problem as they are guaranteed to be non-blocking.

### 3.5.4. Non-blocking content transfer

The process of content transfer for non-blocking connections works completely differently compared to that of blocking connections, as non-blocking connections need to accommodate to the asynchronous nature of the NIO model. The main distinction between two types of connections is inability to use the usual, but inherently blocking java.io.InputStream and java.io.OutputStream classes to represent streams of inbound and outbound content. HttpCore NIO provides ContentEncoder and ContentDecoder interfaces to handle the process of asynchronous content transfer. Non-blocking HTTP connections will instantiate the appropriate implementation of a content codec based on properties of the entity enclosed with the message.

Non-blocking HTTP connections will fire input events until the content entity is fully transferred.

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

- Content-Length **delimited**: The end of the content entity is determined by the value of the Content-Length header. Maximum entity length: Long#MAX_VALUE.

- **Identity coding**: The end of the content entity is demarcated by closing the underlying connection (end of stream condition). For obvious reasons the identity encoding can only be used on the server side. Max entity length: unlimited.

- **Chunk coding**: The content is sent in small chunks. Max entity length: unlimited.

The appropriate content codec will be created automatically depending on properties of the entity enclosed with the message.

### 3.5.6. Direct channel I/O

Content codes are optimized to read data directly from or write data directly to the underlying I/O session's channel, whenever possible avoiding intermediate buffering in a session buffer. Moreover, those codecs that do not perform any content transformation (Content-Length delimited and identity codecs, for example) can leverage NIO java.nio.FileChannel methods for significantly improved performance of file transfer operations both inbound and outbound.

If the actual content decoder implements FileContentDecoder one can make use of its methods to read incoming content directly to a file bypassing an intermediate java.nio.ByteBuffer.

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

Server side HTTP I/O events as defined by the NHttpServerEventHandler interface:

- connected: Triggered when a new incoming connection has been created.

- requestReceived: Triggered when a new HTTP request is received. The connection passed as a parameter to this method is guaranteed to return a valid HTTP request object. If the request received encloses a request entity this method will be followed a series of inputReady events to transfer the request content.

- inputReady: Triggered when the underlying channel is ready for reading a new portion of the request entity through the corresponding content decoder. If the content consumer is unable to process the incoming content, input event notifications can temporarily suspended using IOControl interface (super interface of NHttpServerConnection). Please note that the NHttpServerConnection and ContentDecoder objects are not thread-safe and should only be used within the context of this method call. The IOControl object can be shared and used on other thread to resume input event notifications when the handler is capable of processing more content.

- responseReady: Triggered when the connection is ready to accept new HTTP response. The protocol handler does not have to submit a response if it is not ready.

- outputReady: Triggered when the underlying channel is ready for writing a next portion of the response entity through the corresponding content encoder. If the content producer is unable to generate the outgoing content, output event notifications can be temporarily suspended using IOControl interface (super interface of NHttpServerConnection). Please note that the NHttpServerConnection and ContentEncoder objects are not thread-safe and should only be used within the context of this method call. The IOControl object can be shared and used on other thread to resume output event notifications when more content is made available.

- exception: Triggered when an I/O error occurrs while reading from or writing to the underlying channel or when an HTTP protocol violation occurs while receiving an HTTP request.

- timeout: Triggered when no input is detected on this connection over the maximum period of inactivity.

- closed: Triggered when the connection has been closed.

Client side HTTP I/O events as defined by the NHttpClientEventHandler interface:

- connected: Triggered when a new outgoing connection has been created. The attachment object passed as a parameter to this event is an arbitrary object that was attached to the session request.

- requestReady: Triggered when the connection is ready to accept new HTTP request. The protocol handler does not have to submit a request if it is not ready.

- outputReady: Triggered when the underlying channel is ready for writing a next portion of the request entity through the corresponding content encoder. If the content producer is unable to generate the outgoing content, output event notifications can be temporarily suspended using IOControl interface (super interface of NHttpClientConnection). Please note that the NHttpClientConnection and ContentEncoder objects are not thread-safe and should only be used within the context of this method call. The IOControl object can be shared and used on other thread to resume output event notifications when more content is made available.

- responseReceived: Triggered when an HTTP response is received. The connection passed as a parameter to this method is guaranteed to return a valid HTTP response object. If the response received encloses a response entity this method will be followed a series of inputReady events to transfer the response content.

- inputReady: Triggered when the underlying channel is ready for reading a new portion of the response entity through the corresponding content decoder. If the content consumer is unable to process the incoming content, input event notifications can be temporarily suspended using IOControl interface (super interface of NHttpClientConnection). Please note that the NHttpClientConnection and ContentDecoder objects are not thread-safe and should only be used within the context of this method call. The IOControl object can be shared and used on other thread to resume input event notifications when the handler is capable of processing more content.

- exception: Triggered when an I/O error occurs while reading from or writing to the underlying channel or when an HTTP protocol violation occurs while receiving an HTTP response.

- timeout: Triggered when no input is detected on this connection over the maximum period of inactivity.

- closed: Triggered when the connection has been closed.

## 3.7. Non-blocking HTTP content producers

As discussed previously the process of content transfer for non-blocking connections works completely differently compared to that for blocking connections. For obvious reasons classic I/O abstraction based on inherently blocking java.io.InputStream and java.io.OutputStream classes is not well suited for asynchronous data transfer. In order to avoid inefficient and potentially blocking I/O operation redirection through java.nio.channels.Channles#newChannel non-blocking HTTP entities are expected to implement NIO specific extension interface HttpAsyncContentProducer.

The HttpAsyncContentProducer interface defines several additional method for efficient streaming of content to a non-blocking HTTP connection:

- produceContent: Invoked to write out a chunk of content to the ContentEncoder . The IOControl interface can be used to suspend output events if the entity is temporarily unable to produce more content. When all content is finished, the producer MUST call ContentEncoder#complete(). Failure to do so may cause the entity to be incorrectly delimited. Please note that the ContentEncoder object is not thread-safe and should only be used within the context of this method call. The IOControl object can be shared and used on other thread resume output event notifications when more content is made available.

- isRepeatable: Determines whether or not this producer is capable of producing its content more than once. Repeatable content producers are expected to be able to recreate their content even after having been closed.

- close: Closes the producer and releases all resources currently allocated by it.

### 3.7.1. Creating non-blocking entities

Several HTTP entity implementations included in HttpCore NIO support HttpAsyncContentProducer interface:

- [NByteArrayEntity]()

- [NStringEntity]()

- [NFileEntity]()

#### 3.7.1.1. NByteArrayEntity

This is a simple self-contained repeatable entity, which receives its content from a given byte array. This byte array is supplied to the constructor.

```
NByteArrayEntity entity = new NByteArrayEntity(new byte[] {1, 2, 3});
```

#### 3.7.1.2. NStringEntity

This is a simple, self-contained, repeatable entity that retrieves its data from a java.lang.String object. It has 2 constructors, one simply constructs with a given string where the other also takes a character encoding for the data in the java.lang.String.

```
NStringEntity myEntity = new NStringEntity("important message",
        Consts.UTF_8);

```

#### 3.7.1.3. NFileEntity

This entity reads its content body from a file. This class is mostly used to stream large files of different types, so one needs to supply the content type of the file to make sure the content can be correctly recognized and processed by the recipient.

```
File staticFile = new File("/path/to/myapp.jar");
NFileEntity entity = new NFileEntity(staticFile,
    ContentType.create("application/java-archive", null));

```

The NHttpEntity will make use of the direct channel I/O whenever possible, provided the content encoder is capable of transferring data directly from a file to the socket of the underlying connection.

## 3.8. Non-blocking HTTP protocol handlers

### 3.8.1. Asynchronous HTTP service

HttpAsyncService is a fully asynchronous HTTP server side protocol handler based on the non-blocking (NIO) I/O model. HttpAsyncService translates individual events fired through the NHttpServerEventHandler interface into logically related HTTP message exchanges.

Upon receiving an incoming request the HttpAsyncService verifies the message for compliance with the server expectations using HttpAsyncExpectationVerifier, if provided, and then HttpAsyncRequestHandlerResolver is used to resolve the request URI to a particular HttpAsyncRequestHandler intended to handle the request with the given URI. The protocol handler uses the selected HttpAsyncRequestHandler instance to process the incoming request and to generate an outgoing response.

HttpAsyncService relies on HttpProcessor to generate mandatory protocol headers for all outgoing messages and apply common, cross-cutting message transformations to all incoming and outgoing messages, whereas individual HTTP request handlers are expected to implement application specific content generation and processing.

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

#### 3.8.1.2. Asynchronous HTTP exchange

The most fundamental difference of the non-blocking request handlers compared to their blocking counterparts is ability to defer transmission of the HTTP response back to the client without blocking the I/O thread by delegating the process of handling the HTTP request to a worker thread or another service. The instance of HttpAsyncExchange passed as a parameter to the HttpAsyncRequestHandler#handle method to submit a response as at a later point once response content becomes available.

The HttpAsyncExchange interface can be interacted with using the following methods:

- getRequest: Returns the received HTTP request message.

- getResponse: Returns the default HTTP response message that can submitted once ready.

- submitResponse: Submits an HTTP response and completed the message exchange.

- isCompleted: Determines whether or not the message exchange has been completed.

- setCallback: Sets Cancellable callback to be invoked in case the underlying connection times out or gets terminated prematurely by the client. This callback can be used to cancel a long running response generating process if a response is no longer needed.

- setTimeout: Sets timeout for this message exchange.

- getTimeout: Returns timeout for this message exchange.

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

#### 3.8.1.3. Asynchronous HTTP request consumer

HttpAsyncRequestConsumer facilitates the process of asynchronous processing of HTTP requests. It is a callback interface used by HttpAsyncRequestHandlers to process an incoming HTTP request message and to stream its content from a non-blocking server side HTTP connection.

HTTP I/O events and methods as defined by the HttpAsyncRequestConsumer interface:

- requestReceived: Invoked when a HTTP request message is received.

- consumeContent: Invoked to process a chunk of content from the ContentDecoder. The IOControl interface can be used to suspend input events if the consumer is temporarily unable to consume more content. The consumer can use the ContentDecoder#isCompleted() method to find out whether or not the message content has been fully consumed. Please note that the ContentDecoder object is not thread-safe and should only be used within the context of this method call. The IOControl object can be shared and used on other thread to resume input event notifications when the consumer is capable of processing more content. This event is invoked only if the incoming request message has a content entity enclosed in it.

- requestCompleted: Invoked to signal that the request has been fully processed.

- failed: Invoked to signal that the request processing terminated abnormally.

- getException: Returns an exception in case of an abnormal termination. This method returns null if the request execution is still ongoing or if it completed successfully.

- getResult: Returns a result of the request execution, when available. This method returns null if the request execution is still ongoing.

- isDone: Determines whether or not the request execution completed. If the request processing terminated normally getResult() can be used to obtain the result. If the request processing terminated abnormally getException() can be used to obtain the cause.

- close: Closes the consumer and releases all resources currently allocated by it.

HttpAsyncRequestConsumer implementations are expected to be thread-safe.

BasicAsyncRequestConsumer is a very basic implementation of the HttpAsyncRequestConsumer interface shipped with the library. Please note that this consumer buffers request content in memory and therefore should be used for relatively small request messages.

#### 3.8.1.4. Asynchronous HTTP response producer

HttpAsyncResponseProducer facilitates the process of asynchronous generation of HTTP responses. It is a callback interface used by HttpAsyncRequestHandlers to generate an HTTP response message and to stream its content to a non-blocking server side HTTP connection.

HTTP I/O events and methods as defined by the HttpAsyncResponseProducer interface:

- generateResponse: Invoked to generate a HTTP response message header.

- produceContent: Invoked to write out a chunk of content to the ContentEncoder. The IOControl interface can be used to suspend output events if the producer is temporarily unable to produce more content. When all content is finished, the producer MUST call ContentEncoder#complete(). Failure to do so may cause the entity to be incorrectly delimited. Please note that the ContentEncoder object is not thread-safe and should only be used within the context of this method call. The IOControl object can be shared and used on other thread resume output event notifications when more content is made available. This event is invoked only for if the outgoing response message has a content entity enclosed in it, that is HttpResponse#getEntity() returns null.

- responseCompleted: Invoked to signal that the response has been fully written out.

- failed: Invoked to signal that the response processing terminated abnormally.

- close: Closes the producer and releases all resources currently allocated by it.

HttpAsyncResponseProducer implementations are expected to be thread-safe.

BasicAsyncResponseProducer is a basic implementation of the HttpAsyncResponseProducer interface shipped with the library. The producer can make use of the HttpAsyncContentProducer interface to efficiently stream out message content to a non-blocking HTTP connection, if it is implemented by the HttpEntity enclosed in the response.

#### 3.8.1.5. Non-blocking request handler resolver

The management of non-blocking HTTP request handlers is quite similar to that of blocking HTTP request handlers. Usually an instance of HttpAsyncRequestHandlerResolver is used to maintain a registry of request handlers and to matches a request URI to a particular request handler. HttpCore includes only a very simple implementation of the request handler resolver based on a trivial pattern matching algorithm: HttpAsyncRequestHandlerRegistry supports only three formats: _, <uri>_ and \*<uri>.

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

### 3.8.2. Asynchronous HTTP request executor

HttpAsyncRequestExecutor is a fully asynchronous client side HTTP protocol handler based on the NIO (non-blocking) I/O model. HttpAsyncRequestExecutor translates individual events fired through the NHttpClientEventHandler interface into logically related HTTP message exchanges.

HttpAsyncRequestExecutor relies on HttpAsyncRequestExecutionHandler to implement application specific content generation and processing and to handle logically related series of HTTP request / response exchanges, which may also span across multiple connections. HttpProcessor provided by the HttpAsyncRequestExecutionHandler instance will be used to generate mandatory protocol headers for all outgoing messages and apply common, cross-cutting message transformations to all incoming and outgoing messages. The caller is expected to pass an instance of HttpAsyncRequestExecutionHandler to be used for the next series of HTTP message exchanges through the connection context using HttpAsyncRequestExecutor#HTTP_HANDLER attribute. HTTP exchange sequence is considered complete when the HttpAsyncRequestExecutionHandler#isDone() method returns true.

```
HttpAsyncRequestExecutor ph = new HttpAsyncRequestExecutor();
IOEventDispatch ioEventDispatch = new DefaultHttpClientIODispatch(ph,
        new DefaultNHttpClientConnectionFactory(ConnectionConfig.DEFAULT));
ConnectingIOReactor ioreactor = new DefaultConnectingIOReactor();
ioreactor.execute(ioEventDispatch);
```

The HttpAsyncRequester utility class can be used to abstract away low level details of HttpAsyncRequestExecutionHandler management. Please note HttpAsyncRequester supports single HTTP request / response exchanges only. It does not support HTTP authentication and does not handle redirects automatically.

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

- produceContent: Invoked to write out a chunk of content to the ContentEncoder. The IOControl interface can be used to suspend output events if the producer is temporarily unable to produce more content. When all content is finished, the producer MUST call ContentEncoder#complete(). Failure to do so may cause the entity to be incorrectly delimited Please note that the ContentEncoder object is not thread-safe and should only be used within the context of this method call. The IOControl object can be shared and used on other thread resume output event notifications when more content is made available. This event is invoked only for if the outgoing request message has a content entity enclosed in it, that is HttpEntityEnclosingRequest#getEntity() returns null .

- requestCompleted: Invoked to signal that the request has been fully written out.

- failed: Invoked to signal that the request processing terminated abnormally.

- resetRequest: Invoked to reset the producer to its initial state. Repeatable request producers are expected to release currently allocated resources that are no longer needed or re-acquire resources needed to repeat the process.

- close: Closes the producer and releases all resources currently allocated by it.

HttpAsyncRequestProducer implementations are expected to be thread-safe.

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

## 3.9. Non-blocking connection pools

Non-blocking connection pools are quite similar to blocking one with one significant distinction that they have to reply an I/O reactor to establish new connections. As a result connections leased from a non-blocking pool are returned fully initialized and already bound to a particular I/O session. Non-blocking connections managed by a connection pool cannot be bound to an arbitrary I/O session.

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

3.10. Pipelined request execution

In addition to the normal request / response execution mode HttpAsyncRequester is also capable of executing requests in the so called pipelined mode whereby several requests are immediately written out to the underlying connection. Please note that entity enclosing requests can be executed in the pipelined mode but the 'expect: continue' handshake should be disabled (request messages should contains no 'Expect: 100-continue' header).

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

## 3.11. Non-blocking TLS/SSL

### 3.11.1. SSL I/O session

SSLIOSession is a decorator class intended to transparently extend any arbitrary IOSession with transport layer security capabilities based on the SSL/TLS protocol. Default HTTP connection implementations and protocol handlers should be able to work with SSL sessions without special preconditions or modifications.

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

SSL events as defined by the SSLSetupHandler interface:

- initalize: Triggered when the SSL connection is being initialized. The handler can use this callback to customize properties of the javax.net.ssl.SSLEngine used to establish the SSL session.

- verify: Triggered when the SSL connection has been established and initial SSL handshake has been successfully completed. The handler can use this callback to verify properties of the SSLSession. For instance this would be the right place to enforce SSL cipher strength, validate certificate chain and do hostname checks.

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

- The I/O dispatch may need to call SSLIOSession#initalize() method in order to put the SSL session either into a client or a server mode, if the SSL session has not been yet initialized.

- When the underlying I/O session is input ready, the I/O dispatcher should check whether the SSL I/O session is ready to produce input data by calling SSLIOSession#isAppInputReady(), pass control to the protocol handler if it is, and finally call SSLIOSession#inboundTransport() method in order to do the necessary SSL handshaking and decrypt input data.

- When the underlying I/O session is output ready, the I/O dispatcher should check whether the SSL I/O session is ready to accept output data by calling SSLIOSession#isAppOutputReady(), pass control to the protocol handler if it is, and finally call SSLIOSession#outboundTransport() method in order to do the necessary SSL handshaking and encrypt application data.

## 3.12. Embedded non-blocking HTTP server

As of version 4.4 HttpCore ships with an embedded non-blocking HTTP server based on non-blocking I/O components described above.

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
