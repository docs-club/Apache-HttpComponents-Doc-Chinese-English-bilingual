# Chapter 4. Advanced topics

## 4.1. HTTP message parsing and formatting framework

HTTP message processing framework is designed to be expressive and flexible while remaining memory efficient and fast. HttpCore HTTP message processing code achieves near zero intermediate garbage and near zero-copy buffering for its parsing and formatting operations. The same HTTP message parsing and formatting API and implementations are used by both the blocking and non-blocking transport implementations, which helps ensure a consistent behavior of HTTP services regardless of the I/O model.

### 4.1.1. HTTP line parsing and formatting

HttpCore utilizes a number of low level components for all its line parsing and formatting methods.

CharArrayBuffer represents a sequence of characters, usually a single line in an HTTP message stream such as a request line, a status line or a header. Internally CharArrayBuffer is backed by an array of chars, which can be expanded to accommodate more input if needed. CharArrayBuffer also provides a number of utility methods for manipulating content of the buffer, storing more data and retrieving subsets of data.

```
CharArrayBuffer buf = new CharArrayBuffer(64);
buf.append("header:  data ");
int i = buf.indexOf(':');
String s = buf.substringTrimmed(i + 1, buf.length());
System.out.println(s);
System.out.println(s.length());
```

stdout >

```
data
4
```

ParserCursor represents a context of a parsing operation: the bounds limiting the scope of the parsing operation and the current position the parsing operation is expected to start at.

```
CharArrayBuffer buf = new CharArrayBuffer(64);
buf.append("header:  data ");
int i = buf.indexOf(':');
ParserCursor cursor = new ParserCursor(0, buf.length());
cursor.updatePos(i + 1);
System.out.println(cursor);
```

stdout >

```
[0>7>14]
```

LineParser is the interface for parsing lines in the head section of an HTTP message. There are individual methods for parsing a request line, a status line, or a header line. The lines to parse are passed in-memory, the parser does not depend on any specific I/O mechanism.

```
CharArrayBuffer buf = new CharArrayBuffer(64);
buf.append("HTTP/1.1 200");
ParserCursor cursor = new ParserCursor(0, buf.length());

LineParser parser = BasicLineParser.INSTANCE;
ProtocolVersion ver = parser.parseProtocolVersion(buf, cursor);
System.out.println(ver);
System.out.println(buf.substringTrimmed(
        cursor.getPos(),
        cursor.getUpperBound()));
```

stdout >

```
HTTP/1.1
200
```

```
CharArrayBuffer buf = new CharArrayBuffer(64);
buf.append("HTTP/1.1 200 OK");
ParserCursor cursor = new ParserCursor(0, buf.length());
LineParser parser = new BasicLineParser();
StatusLine sl = parser.parseStatusLine(buf, cursor);
System.out.println(sl.getReasonPhrase());
```

stdout >

```
OK
```

LineFormatter for formatting elements of the head section of an HTTP message. This is the complement to LineParser . There are individual methods for formatting a request line, a status line, or a header line.

Please note the formatting does not include the trailing line break sequence CR-LF.

```
CharArrayBuffer buf = new CharArrayBuffer(64);
LineFormatter formatter = new BasicLineFormatter();
formatter.formatRequestLine(buf,
    new BasicRequestLine("GET", "/", HttpVersion.HTTP_1_1));
System.out.println(buf.toString());
formatter.formatHeader(buf,
    new BasicHeader("Content-Type", "text/plain"));
System.out.println(buf.toString());
```

stdout >

```
GET / HTTP/1.1
Content-Type: text/plain
```

HeaderValueParser is the interface for parsing header values into elements.

```
CharArrayBuffer buf = new CharArrayBuffer(64);
HeaderValueParser parser = new BasicHeaderValueParser();
buf.append("name1=value1; param1=p1, " +
    "name2 = \"value2\", name3  = value3");
ParserCursor cursor = new ParserCursor(0, buf.length());
System.out.println(parser.parseHeaderElement(buf, cursor));
System.out.println(parser.parseHeaderElement(buf, cursor));
System.out.println(parser.parseHeaderElement(buf, cursor));
```

stdout >

```
name1=value1; param1=p1
name2=value2
name3=value3
```

HeaderValueFormatter is the interface for formatting elements of a header value. This is the complement to HeaderValueParser .

```
CharArrayBuffer buf = new CharArrayBuffer(64);
HeaderValueFormatter formatter = new BasicHeaderValueFormatter();
HeaderElement[] hes = new HeaderElement[] {
        new BasicHeaderElement("name1", "value1",
                new NameValuePair[] {
                    new BasicNameValuePair("param1", "p1")} ),
        new BasicHeaderElement("name2", "value2"),
        new BasicHeaderElement("name3", "value3"),
};
formatter.formatElements(buf, hes, true);
System.out.println(buf.toString());
```

stdout >

```
name1="value1"; param1="p1", name2="value2", name3="value3"
```

### 4.1.2. HTTP message streams and session I/O buffers

HttpCore provides a number of utility classes for the blocking and non-blocking I/O models that facilitate the processing of HTTP message streams, simplify handling of CR-LF delimited lines in HTTP messages and manage intermediate data buffering.

HTTP connection implementations usually rely on session input/output buffers for reading and writing data from and to an HTTP message stream. Session input/output buffer implementations are I/O model specific and are optimized either for blocking or non-blocking operations.

Blocking HTTP connections use socket bound session buffers to transfer data. Session buffer interfaces are similar to java.io.InputStream / java.io.OutputStream classes, but they also provide methods for reading and writing CR-LF delimited lines.

```
Socket socket1 = <...>
Socket socket2 = <...>
HttpTransportMetricsImpl metrics = new HttpTransportMetricsImpl();
SessionInputBufferImpl inbuffer = new SessionInputBufferImpl(metrics, 8 * 1024);
inbuffer.bind(socket1.getInputStream());
SessionOutputBufferImpl outbuffer = new SessionOutputBufferImpl(metrics, 8 * 1024);
outbuffer.bind(socket2.getOutputStream());
CharArrayBuffer linebuf = new CharArrayBuffer(1024);
inbuffer.readLine(linebuf);
outbuffer.writeLine(linebuf);
```

Non-blocking HTTP connections use session buffers optimized for reading and writing data from and to non-blocking NIO channels. NIO session input/output sessions help deal with CR-LF delimited lines in a non-blocking I/O mode.

```
ReadableByteChannel channel1 = <...>
WritableByteChannel channel2 = <...>

SessionInputBuffer inbuffer = new SessionInputBufferImpl(8 * 1024);
SessionOutputBuffer outbuffer = new SessionOutputBufferImpl(8 * 1024);

CharArrayBuffer linebuf = new CharArrayBuffer(1024);
boolean endOfStream = false;
int bytesRead = inbuffer.fill(channel1);
if (bytesRead == -1) {
    endOfStream = true;
}
if (inbuffer.readLine(linebuf, endOfStream)) {
    outbuffer.writeLine(linebuf);
}
if (outbuffer.hasData()) {
    outbuffer.flush(channel2);
}
```

### 4.1.3. HTTP message parsers and formatters

HttpCore also provides coarse-grained facade type interfaces for parsing and formatting of HTTP messages. Default implementations of those interfaces build upon the functionality provided by SessionInputBuffer / SessionOutputBuffer and HttpLineParser / HttpLineFormatter implementations.

Example of HTTP request parsing / writing for blocking HTTP connections:

```
SessionInputBuffer inbuffer = <...>
SessionOutputBuffer outbuffer = <...>

HttpMessageParser<HttpRequest> requestParser = new DefaultHttpRequestParser(
    inbuffer);
HttpRequest request = requestParser.parse();
HttpMessageWriter<HttpRequest> requestWriter = new DefaultHttpRequestWriter(
    outbuffer);
requestWriter.write(request);
```

Example of HTTP response parsing / writing for blocking HTTP connections:

```
SessionInputBuffer inbuffer = <...>
SessionOutputBuffer outbuffer = <...>

HttpMessageParser<HttpResponse> responseParser = new DefaultHttpResponseParser(
        inbuffer);
HttpResponse response = responseParser.parse();
HttpMessageWriter<HttpResponse> responseWriter = new DefaultHttpResponseWriter(
        outbuffer);
responseWriter.write(response);
```

Custom message parsers and writers can be plugged into the message processing pipeline through a custom connection factory:

```
HttpMessageWriterFactory<HttpResponse> responseWriterFactory =
                                new HttpMessageWriterFactory<HttpResponse>() {
    @Override
    public HttpMessageWriter<HttpResponse> create(
            SessionOutputBuffer buffer) {
        HttpMessageWriter<HttpResponse> customWriter = <...>
        return customWriter;
    }
};
HttpMessageParserFactory<HttpRequest> requestParserFactory =
                                new HttpMessageParserFactory<HttpRequest>() {
    @Override
    public HttpMessageParser<HttpRequest> create(
            SessionInputBuffer buffer,
            MessageConstraints constraints) {
        HttpMessageParser<HttpRequest> customParser = <...>
        return customParser;
    }
};
HttpConnectionFactory<DefaultBHttpServerConnection> cf =
                                new DefaultBHttpServerConnectionFactory(
        ConnectionConfig.DEFAULT,
        requestParserFactory,
        responseWriterFactory);
Socket socket = <...>
DefaultBHttpServerConnection conn = cf.createConnection(socket);
```

Example of HTTP request parsing / writing for non-blocking HTTP connections:

```
SessionInputBuffer inbuffer = <...>
SessionOutputBuffer outbuffer  = <...>

NHttpMessageParser<HttpRequest> requestParser = new DefaultHttpRequestParser(
        inbuffer);
HttpRequest request = requestParser.parse();
NHttpMessageWriter<HttpRequest> requestWriter = new DefaultHttpRequestWriter(
        outbuffer);
requestWriter.write(request);
```

Example of HTTP response parsing / writing for non-blocking HTTP connections:

```
SessionInputBuffer inbuffer = <...>
SessionOutputBuffer outbuffer  = <...>

NHttpMessageParser<HttpResponse> responseParser = new DefaultHttpResponseParser(
        inbuffer);
HttpResponse response = responseParser.parse();
NHttpMessageWriter responseWriter = new DefaultHttpResponseWriter(
        outbuffer);
responseWriter.write(response);
```

Custom non-blocking message parsers and writers can be plugged into the message processing pipeline through a custom connection factory:

```
NHttpMessageWriterFactory<HttpResponse> responseWriterFactory =
                        new NHttpMessageWriterFactory<HttpResponse>() {
    @Override
    public NHttpMessageWriter<HttpResponse> create(SessionOutputBuffer buffer) {
        NHttpMessageWriter<HttpResponse> customWriter = <...>
        return customWriter;
    }
};
NHttpMessageParserFactory<HttpRequest> requestParserFactory =
                        new NHttpMessageParserFactory<HttpRequest>() {
    @Override
    public NHttpMessageParser<HttpRequest> create(
            SessionInputBuffer buffer, MessageConstraints constraints) {
        NHttpMessageParser<HttpRequest> customParser = <...>
        return customParser;
    }
};
NHttpConnectionFactory<DefaultNHttpServerConnection> cf =
                        new DefaultNHttpServerConnectionFactory(
        null,
        requestParserFactory,
        responseWriterFactory,
        ConnectionConfig.DEFAULT);
IOSession iosession = <...>
DefaultNHttpServerConnection conn = cf.createConnection(iosession);
```

### 4.1.4. HTTP header parsing on demand

The default implementations of HttpMessageParser and NHttpMessageParser interfaces do not parse HTTP headers immediately. Parsing of header value is deferred until its properties are accessed. Those headers that are never used by the application will not be parsed at all. The CharArrayBuffer backing the header can be obtained through an optional FormattedHeader interface.

```
HttpResponse response = <...>
Header h1 = response.getFirstHeader("Content-Type");
if (h1 instanceof FormattedHeader) {
    CharArrayBuffer buf = ((FormattedHeader) h1).getBuffer();
    System.out.println(buf);
}
```
