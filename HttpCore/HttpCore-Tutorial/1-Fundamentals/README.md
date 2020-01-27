# Chapter 1. Fundamentals（基础）

## 1.1. HTTP messages（HTTP 消息）

### 1.1.1. Structure（结构）

A HTTP message consists of a header and an optional body. The message header of an HTTP request consists of a request line and a collection of header fields. The message header of an HTTP response consists of a status line and a collection of header fields. All HTTP messages must include the protocol version. Some HTTP messages can optionally enclose a content body.

HTTP 消息由一个头和一个可选的主体组成。HTTP 请求的消息头由请求行和头字段集合组成。HTTP 响应的消息头由一个状态行和一个头字段集合组成。所有 HTTP 消息必须包含协议版本。一些 HTTP 消息可以选择封装内容主体。

HttpCore defines the HTTP message object model to follow this definition closely, and provides extensive support for serialization (formatting) and deserialization (parsing) of HTTP message elements.

HttpCore 定义了 HTTP 消息对象模型来严格遵循这个定义，并提供了对 HTTP 消息元素的序列化（格式化）和反序列化（解析）的广泛支持。

### 1.1.2. Basic operations（基本操作）

#### 1.1.2.1. HTTP request message（HTTP 请求消息）

HTTP request is a message sent from the client to the server. The first line of that message includes the method to apply to the resource, the identifier of the resource, and the protocol version in use.

HTTP 请求是从客户端发送到服务器的消息。该消息的第一行包括应用于资源的方法、资源的标识符和正在使用的协议版本。

```
HttpRequest request = new BasicHttpRequest("GET", "/",
    HttpVersion.HTTP_1_1);

System.out.println(request.getRequestLine().getMethod());
System.out.println(request.getRequestLine().getUri());
System.out.println(request.getProtocolVersion());
System.out.println(request.getRequestLine().toString());
```

stdout >

```
GET
/
HTTP/1.1
GET / HTTP/1.1
```

#### 1.1.2.2. HTTP response message（HTTP 响应消息）

**注：该部分译文参见 [HttpClient 1.1.2]()**

HTTP response is a message sent by the server back to the client after having received and interpreted a request message. The first line of that message consists of the protocol version followed by a numeric status code and its associated textual phrase.

```
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1,
    HttpStatus.SC_OK, "OK");

System.out.println(response.getProtocolVersion());
System.out.println(response.getStatusLine().getStatusCode());
System.out.println(response.getStatusLine().getReasonPhrase());
System.out.println(response.getStatusLine().toString());
```

stdout >

```
HTTP/1.1
200
OK
HTTP/1.1 200 OK
```

#### 1.1.2.3. HTTP message common properties and methods（HTTP 消息的公共属性和方法）

An HTTP message can contain a number of headers describing properties of the message such as the content length, content type, and so on. HttpCore provides methods to retrieve, add, remove, and enumerate such headers.

注：译文参见 [HttpClient 1.1.3]()

```
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1,
    HttpStatus.SC_OK, "OK");
response.addHeader("Set-Cookie",
    "c1=a; path=/; domain=localhost");
response.addHeader("Set-Cookie",
    "c2=b; path=\"/\", c3=c; domain=\"localhost\"");
Header h1 = response.getFirstHeader("Set-Cookie");
System.out.println(h1);
Header h2 = response.getLastHeader("Set-Cookie");
System.out.println(h2);
Header[] hs = response.getHeaders("Set-Cookie");
System.out.println(hs.length);
```

stdout >

```
Set-Cookie: c1=a; path=/; domain=localhost
Set-Cookie: c2=b; path="/", c3=c; domain="localhost"
2
```

There is an efficient way to obtain all headers of a given type using the HeaderIterator interface.

**注：译文参见 [HttpClient 1.1.3]()，仅开头用词差异，语义相同。**

```
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1,
    HttpStatus.SC_OK, "OK");
response.addHeader("Set-Cookie",
    "c1=a; path=/; domain=localhost");
response.addHeader("Set-Cookie",
    "c2=b; path=\"/\", c3=c; domain=\"localhost\"");

HeaderIterator it = response.headerIterator("Set-Cookie");

while (it.hasNext()) {
    System.out.println(it.next());
}
```

stdout >

```
Set-Cookie: c1=a; path=/; domain=localhost
Set-Cookie: c2=b; path="/", c3=c; domain="localhost"
```

It also provides convenience methods to parse HTTP messages into individual header elements.

**注：译文参见 [HttpClient 1.1.3]()**

```
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1,
    HttpStatus.SC_OK, "OK");
response.addHeader("Set-Cookie",
    "c1=a; path=/; domain=localhost");
response.addHeader("Set-Cookie",
    "c2=b; path=\"/\", c3=c; domain=\"localhost\"");

HeaderElementIterator it = new BasicHeaderElementIterator(
        response.headerIterator("Set-Cookie"));

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

```
c1 = a
 path=/
 domain=localhost
c2 = b
 path=/
c3 = c
 domain=localhost
```

HTTP headers are tokenized into individual header elements only on demand. HTTP headers received over an HTTP connection are stored internally as an array of characters and parsed lazily only when you access their properties.

HTTP 头在有必要时可以分割成单个元素。通过 HTTP 连接接收到的 HTTP 报头在内部存储为字符数组，只有在访问它们的属性时才进行延迟解析。

### 1.1.3. HTTP entity（HTTP 实体）

HTTP messages can carry a content entity associated with the request or response. Entities can be found in some requests and in some responses, as they are optional. Requests that use entities are referred to as entity-enclosing requests. The HTTP specification defines two entity-enclosing methods: POST and PUT. Responses are usually expected to enclose a content entity. There are exceptions to this rule such as responses to HEAD method and 204 No Content, 304 Not Modified, 205 Reset Content responses.

**注：译文参见 [HttpClient 1.1.4]()**

HttpCore distinguishes three kinds of entities, depending on where their content originates:

**注：译文参见 [HttpClient 1.1.4]()**

- **streamed**: The content is received from a stream, or generated on the fly. In particular, this category includes entities being received from a connection. Streamed entities are generally not repeatable.

**注：译文参见 [HttpClient 1.1.4]()**

- **self-contained**: The content is in memory or obtained by means that are independent from a connection or other entity. Self-contained entities are generally repeatable.

**注：译文参见 [HttpClient 1.1.4]()**

- **wrapping**: The content is obtained from another entity.

**注：译文参见 [HttpClient 1.1.4]()**

#### 1.1.3.1. Repeatable entities

An entity can be repeatable, meaning its content can be read more than once. This is only possible with self-contained entities (like ByteArrayEntity or StringEntity).

**注：译文参见 [HttpClient 1.1.4.1]()**

#### 1.1.3.2. Using HTTP entities

Since an entity can represent both binary and character content, it has support for character encodings (to support the latter, i.e. character content).

**注：译文参见 [HttpClient 1.1.4.2]()第一段**

The entity is created when executing a request with enclosed content or when the request was successful and the response body is used to send the result back to the client.

**注：译文参见 [HttpClient 1.1.4.2]()第二段**

To read the content from the entity, one can either retrieve the input stream via the HttpEntity#getContent() method, which returns an java.io.InputStream, or one can supply an output stream to the HttpEntity#writeTo(OutputStream) method, which will return once all content has been written to the given stream. Please note that some non-streaming (self-contained) entities may be unable to represent their content as a java.io.InputStream efficiently. It is legal for such entities to implement HttpEntity#writeTo(OutputStream) method only and to throw UnsupportedOperationException from HttpEntity#getContent() method.

**注："To read...the given stream."的译文参见 [HttpClient 1.1.4.2]()第三段，剩余部分如下：**

请注意，一些非流(自包含的)实体可能无法将其内容表示为 java.io。InputStream 有效。这类实体只实现 HttpEntity#writeTo(OutputStream)方法，并从 HttpEntity#getContent()方法抛出 UnsupportedOperationException 是合法的。

The EntityUtils class exposes several static methods to simplify extracting the content or information from an entity. Instead of reading the java.io.InputStream directly, one can retrieve the complete content body in a string or byte array by using the methods from this class.

**注：译文参见 [HttpClient 1.1.6]()第一段，语义相同**

When the entity has been received with an incoming message, the methods HttpEntity#getContentType() and HttpEntity#getContentLength() methods can be used for reading the common metadata such as Content-Type and Content-Length headers (if they are available). Since the Content-Type header can contain a character encoding for text mime-types like text/plain or text/html, the HttpEntity#getContentEncoding() method is used to read this information. If the headers aren't available, a length of -1 will be returned, and NULL for the content type. If the Content-Type header is available, a Header object will be returned.

**注：译文参见 [HttpClient 1.1.4.2]()第四段**

When creating an entity for a outgoing message, this meta data has to be supplied by the creator of the entity.

**注：译文参见 [HttpClient 1.1.4.2]()第五段**

```
StringEntity myEntity = new StringEntity("important message",
    Consts.UTF_8);

System.out.println(myEntity.getContentType());
System.out.println(myEntity.getContentLength());
System.out.println(EntityUtils.toString(myEntity));
System.out.println(EntityUtils.toByteArray(myEntity).length);
```

stdout >

```
Content-Type: text/plain; charset=UTF-8
17
important message
17
```

#### 1.1.3.3. Ensuring release of system resources

In order to ensure proper release of system resources one must close the content stream associated with the entity.

**注：译文参见 [HttpClient 1.1.5]()第一段**

```
HttpResponse response;
HttpEntity entity = response.getEntity();
if (entity != null) {
    InputStream instream = entity.getContent();
    try {
        // do something useful
    } finally {
        instream.close();
    }
}
```

When working with streaming entities, one can use the EntityUtils#consume(HttpEntity) method to ensure that the entity content has been fully consumed and the underlying stream has been closed.

**注：译文参见 [HttpClient 1.1.5]()第四段**

### 1.1.4. Creating entities

There are a few ways to create entities. HttpCore provides the following implementations:

有几种创建实体的方法。HttpCore 提供了以下实现：

- BasicHttpEntity
- ByteArrayEntity
- StringEntity
- InputStreamEntity
- FileEntity
- EntityTemplate
- HttpEntityWrapper
- BufferedHttpEntity

#### 1.1.4.1. BasicHttpEntity

Exactly as the name implies, this basic entity represents an underlying stream. In general, use this class for entities received from HTTP messages.

顾名思义，这个基础实体表示底层流。通常，对于从 HTTP 消息接收到的实体，使用这个类。

This entity has an empty constructor. After construction, it represents no content, and has a negative content length.

这个实体有一个空的构造函数。在构造之后，它表示没有内容，并且具有负的内容长度。

One needs to set the content stream, and optionally the length. This can be done with the BasicHttpEntity#setContent(InputStream) and BasicHttpEntity#setContentLength(long) methods respectively.

需要设置内容流和长度（可选）。这可以分别使用 BasicHttpEntity#setContent(InputStream) 和 BasicHttpEntity#setContentLength(long) 方法来完成。

```
BasicHttpEntity myEntity = new BasicHttpEntity();
myEntity.setContent(someInputStream);
myEntity.setContentLength(340); // sets the length to 340
```

#### 1.1.4.2. ByteArrayEntity

ByteArrayEntity is a self-contained, repeatable entity that obtains its content from a given byte array. Supply the byte array to the constructor.

ByteArrayEntity 是一个 self-contained、可重复的实体，它从给定的字节数组中获取内容。将字节数组提供给构造函数。

```
ByteArrayEntity myEntity = new ByteArrayEntity(new byte[] {1,2,3},
ContentType.APPLICATION_OCTET_STREAM);
```

1.1.4.3. StringEntity

StringEntity is a self-contained, repeatable entity that obtains its content from a java.lang.String object. It has three constructors, one simply constructs with a given java.lang.String object; the second also takes a character encoding for the data in the string; the third allows the mime type to be specified.

StringEntity 是一个 self-contained、可重复的实体，它从 java.lang.String 获取其内容。它有三个构造函数，其中一个只是用给定的 java.lang.String 构造；第二种方法对字符串中的数据进行字符编码；第三种方法允许指定 mime 类型。

```
StringBuilder sb = new StringBuilder();
Map<String, String> env = System.getenv();
for (Map.Entry<String, String> envEntry : env.entrySet()) {
    sb.append(envEntry.getKey())
            .append(": ").append(envEntry.getValue())
            .append("\r\n");
}

// construct without a character encoding (defaults to ISO-8859-1)
HttpEntity myEntity1 = new StringEntity(sb.toString());

// alternatively construct with an encoding (mime type defaults to "text/plain")
HttpEntity myEntity2 = new StringEntity(sb.toString(), Consts.UTF_8);

// alternatively construct with an encoding and a mime type
HttpEntity myEntity3 = new StringEntity(sb.toString(),
        ContentType.create("text/plain", Consts.UTF_8));
```

#### 1.1.4.4. InputStreamEntity

InputStreamEntity is a streamed, non-repeatable entity that obtains its content from an input stream. Construct it by supplying the input stream and the content length. Use the content length to limit the amount of data read from the java.io.InputStream. If the length matches the content length available on the input stream, then all data will be sent. Alternatively, a negative content length will read all data from the input stream, which is the same as supplying the exact content length, so use the length to limit the amount of data to read.

InputStreamEntity 是一个基于流的、不可重复的实体，它从输入流中获取内容。通过提供输入流和内容长度来构造它。使用内容长度来限制从 java.io.InputStream 中读取的数据量。如果长度与输入流中可用的内容长度匹配，则将发送所有数据。或者，负的内容长度将从输入流读取所有数据，这与提供准确的内容长度相同，因此使用长度来限制要读取的数据量。

```
InputStream instream = getSomeInputStream();
InputStreamEntity myEntity = new InputStreamEntity(instream, 16);
```

#### 1.1.4.5. FileEntity

FileEntity is a self-contained, repeatable entity that obtains its content from a file. Use this mostly to stream large files of different types, where you need to supply the content type of the file, for instance, sending a zip file would require the content type application/zip, for XML application/xml.

FileEntity 是一个 self-contained、可重复的实体，它从文件中获取内容。这主要用于不同类型的大型文件流，其中需要提供文件的内容类型，例如，发送 zip 文件需要内容类型 application/zip，而发送 XML 则是 application/XML

```
HttpEntity entity = new FileEntity(staticFile,
        ContentType.create("application/java-archive"));
```

#### 1.1.4.6. HttpEntityWrapper

This is the base class for creating wrapped entities. The wrapping entity holds a reference to a wrapped entity and delegates all calls to it. Implementations of wrapping entities can derive from this class and need to override only those methods that should not be delegated to the wrapped entity.

这是创建包装实体的基类。包装实体持有对包装实体的引用，并将所有调用委托给它。包装实体的实现可以从这个类派生，只需要覆盖那些不应该委托给包装实体的方法。

#### 1.1.4.7. BufferedHttpEntity

BufferedHttpEntity is a subclass of HttpEntityWrapper. Construct it by supplying another entity. It reads the content from the supplied entity, and buffers it in memory.

BufferedHttpEntity 是 HttpEntityWrapper 的子类。通过提供另一个实体来构造它。它从提供的实体中读取内容，并缓冲在内存中。

This makes it possible to make a repeatable entity, from a non-repeatable entity. If the supplied entity is already repeatable, it simply passes calls through to the underlying entity.

这使得从一个不可重复的实体变成一个可重复的实体成为可能。如果提供的实体已经是可重复的，则只需将调用传递给底层实体即可。

```
myNonRepeatableEntity.setContent(someInputStream);
BufferedHttpEntity myBufferedEntity = new BufferedHttpEntity(
  myNonRepeatableEntity);
```

## 1.2. HTTP protocol processors（HTTP 协议处理器）

HTTP protocol interceptor is a routine that implements a specific aspect of the HTTP protocol. Usually protocol interceptors are expected to act upon one specific header or a group of related headers of the incoming message or populate the outgoing message with one specific header or a group of related headers. Protocol interceptors can also manipulate content entities enclosed with messages; transparent content compression / decompression being a good example. Usually this is accomplished by using the 'Decorator' pattern where a wrapper entity class is used to decorate the original entity. Several protocol interceptors can be combined to form one logical unit.

**注：译文参见 [HttpClient 1.4]()第一段**

HTTP protocol processor is a collection of protocol interceptors that implements the 'Chain of Responsibility' pattern, where each individual protocol interceptor is expected to work on the particular aspect of the HTTP protocol it is responsible for.

HTTP 协议处理器是实现「责任链」模式的协议拦截器的集合，其中每个独立的协议拦截器都要处理它所负责的 HTTP 协议的特定方面。

Usually the order in which interceptors are executed should not matter as long as they do not depend on a particular state of the execution context. If protocol interceptors have interdependencies and therefore must be executed in a particular order, they should be added to the protocol processor in the same sequence as their expected execution order.

**注：译文参见 [HttpClient 1.4]()第三段**

Protocol interceptors must be implemented as thread-safe. Similarly to servlets, protocol interceptors should not use instance variables unless access to those variables is synchronized.

**注：译文参见 [HttpClient 1.4]()第四段**

### 1.2.1. Standard protocol interceptors

HttpCore comes with a number of most essential protocol interceptors for client and server HTTP processing.

HttpCore 附带了许多用于客户端和服务器 HTTP 处理的最基本的协议拦截器。

#### 1.2.1.1. RequestContent

RequestContent is the most important interceptor for outgoing requests. It is responsible for delimiting content length by adding the Content-Length or Transfer-Content headers based on the properties of the enclosed entity and the protocol version. This interceptor is required for correct functioning of client side protocol processors.

RequestContent 是传出请求最重要的拦截器。它负责根据所包含实体的属性和协议版本，通过添加内容长度或传输内容头来分隔内容长度。这个拦截器是正确运行客户端协议处理器所必需的。

#### 1.2.1.2. ResponseContent

ResponseContent is the most important interceptor for outgoing responses. It is responsible for delimiting content length by adding Content-Length or Transfer-Content headers based on the properties of the enclosed entity and the protocol version. This interceptor is required for correct functioning of server side protocol processors.

ResponseContent 是传出响应最重要的拦截器。它负责根据所包含实体的属性和协议版本，通过添加 Content-Length 或 Transfer-Content 头来分隔内容长度。这个拦截器是服务器端协议处理器正确工作所必需的。

#### 1.2.1.3. RequestConnControl

RequestConnControl is responsible for adding the Connection header to the outgoing requests, which is essential for managing persistence of HTTP/1.0 connections. This interceptor is recommended for client side protocol processors.

RequestConnControl 负责将连接头添加到发出的请求中，这对于管理 HTTP/1.0 连接的持久性至关重要。此拦截器建议用于客户端协议处理器。

#### 1.2.1.4. ResponseConnControl

ResponseConnControl is responsible for adding the Connection header to the outgoing responses, which is essential for managing persistence of HTTP/1.0 connections. This interceptor is recommended for server side protocol processors.

ResponseConnControl 负责将连接头添加到发出的响应中，这对于管理 HTTP/1.0 连接的持久性至关重要。建议将此拦截器用于服务器端协议处理器。

#### 1.2.1.5. RequestDate

RequestDate is responsible for adding the Date header to the outgoing requests. This interceptor is optional for client side protocol processors.

RequestDate 负责向发出的请求添加日期标头。这个拦截器对于客户端协议处理器而言是可选的。

#### 1.2.1.6. ResponseDate

ResponseDate is responsible for adding the Date header to the outgoing responses. This interceptor is recommended for server side protocol processors.

ResponseDate 负责将日期标头添加到发出的响应中。建议将此拦截器用于服务器端协议处理器。

#### 1.2.1.7. RequestExpectContinue

RequestExpectContinue is responsible for enabling the 'expect-continue' handshake by adding the Expect header. This interceptor is recommended for client side protocol processors.

RequestExpectContinue 负责通过添加 Expect 头来启用「expect-continue」握手。建议将此拦截器用于客户端协议处理器。

#### 1.2.1.8. RequestTargetHost

RequestTargetHost is responsible for adding the Host header. This interceptor is required for client side protocol processors.

RequestTargetHost 负责添加主机报头。客户端协议处理器需要这个拦截器。

#### 1.2.1.9. RequestUserAgent

RequestUserAgent is responsible for adding the User-Agent header. This interceptor is recommended for client side protocol processors.

RequestUserAgent 负责添加用户代理头。建议将此拦截器用于客户端协议处理器。

#### 1.2.1.10. ResponseServer

ResponseServer is responsible for adding the Server header. This interceptor is recommended for server side protocol processors.

ResponseServer 负责添加服务器报头。建议将此拦截器用于服务器端协议处理器。

### 1.2.2. Working with protocol processors

Usually HTTP protocol processors are used to pre-process incoming messages prior to executing application specific processing logic and to post-process outgoing messages.

通常，HTTP 协议处理器用于在执行特定于应用程序的处理逻辑之前对传入消息进行预处理，以及对传出消息进行后处理。

```
HttpProcessor httpproc = HttpProcessorBuilder.create()
        // Required protocol interceptors
        .add(new RequestContent())
        .add(new RequestTargetHost())
        // Recommended protocol interceptors
        .add(new RequestConnControl())
        .add(new RequestUserAgent("MyAgent-HTTP/1.1"))
        // Optional protocol interceptors
        .add(new RequestExpectContinue(true))
        .build();

HttpCoreContext context = HttpCoreContext.create();
HttpRequest request = new BasicHttpRequest("GET", "/");
httpproc.process(request, context);
```

Send the request to the target host and get a response.

将请求发送到目标主机并获得响应。

```
HttpResponse = <...>
httpproc.process(response, context);
```

Please note the BasicHttpProcessor class does not synchronize access to its internal structures and therefore may not be thread-safe.

请注意，BasicHttpProcessor 类对其内部结构的访问不会进行同步处理，因此可能不是线程安全的。

## 1.3. HTTP execution context

Originally HTTP has been designed as a stateless, response-request oriented protocol. However, real world applications often need to be able to persist state information through several logically related request-response exchanges. In order to enable applications to maintain a processing state HttpCpre allows HTTP messages to be executed within a particular execution context, referred to as HTTP context. Multiple logically related messages can participate in a logical session if the same context is reused between consecutive requests. HTTP context functions similarly to a `java.util.Map<String, Object>`. It is simply a collection of logically related named values.

**注：译文参见 [HttpClient 1.3]()第一段**

Please nore HttpContext can contain arbitrary objects and therefore may be unsafe to share between multiple threads. Care must be taken to ensure that HttpContext instances can be accessed by one thread at a time.

**注：「HttpContext can...multiple threads.」部分的译文参见 [HttpClient 1.3]()第二段**

### 1.3.1. Context sharing

Protocol interceptors can collaborate by sharing information - such as a processing state - through an HTTP execution context. HTTP context is a structure that can be used to map an attribute name to an attribute value. Internally HTTP context implementations are usually backed by a HashMap. The primary purpose of the HTTP context is to facilitate information sharing among various logically related components. HTTP context can be used to store a processing state for one message or several consecutive messages. Multiple logically related messages can participate in a logical session if the same context is reused between consecutive messages.

**注：「Protocol interceptors...execution context.」部分的译文参见 [HttpClient 1.4]()第一段**

```
HttpProcessor httpproc = HttpProcessorBuilder.create()
        .add(new HttpRequestInterceptor() {
            public void process(
                    HttpRequest request,
                    HttpContext context) throws HttpException, IOException {
                String id = (String) context.getAttribute("session-id");
                if (id != null) {
                    request.addHeader("Session-ID", id);
                }
            }
        })
        .build();

HttpCoreContext context = HttpCoreContext.create();
HttpRequest request = new BasicHttpRequest("GET", "/");
httpproc.process(request, context);
```
