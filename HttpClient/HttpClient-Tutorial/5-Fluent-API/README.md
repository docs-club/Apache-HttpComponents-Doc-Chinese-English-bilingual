# Chapter 5. Fluent API（流畅 API）

## 5.1. Easy to use facade API（易于使用的外观模式 API）

As of version of 4.2 HttpClient comes with an easy to use facade API based on the concept of a fluent interface. Fluent facade API exposes only the most fundamental functions of HttpClient and is intended for simple use cases that do not require the full flexibility of HttpClient. For instance, fluent facade API relieves the users from having to deal with connection management and resource deallocation.

从 4.2 版本开始，HttpClient 提供了一个易于使用的基于流畅接口概念的外观模式API。外观模式 API 只公开 HttpClient 的最基本的功能，用于不需要 HttpClient 的全部灵活性的简单用例。例如，外观模式 API 使用户不必处理连接管理和资源分配。

Here are several examples of HTTP requests executed through the HC fluent API

以下是通过 HC 流畅 API 执行的几个 HTTP 请求示例

```
// Execute a GET with timeout settings and return response content as String.
Request.Get("http://somehost/")
        .connectTimeout(1000)
        .socketTimeout(1000)
        .execute().returnContent().asString();
```

```
// Execute a POST with the 'expect-continue' handshake, using HTTP/1.1,
// containing a request body as String and return response content as byte array.
Request.Post("http://somehost/do-stuff")
        .useExpectContinue()
        .version(HttpVersion.HTTP_1_1)
        .bodyString("Important stuff", ContentType.DEFAULT_TEXT)
        .execute().returnContent().asBytes();
```

```
// Execute a POST with a custom header through the proxy containing a request body
// as an HTML form and save the result to the file
Request.Post("http://somehost/some-form")
        .addHeader("X-Custom-header", "stuff")
        .viaProxy(new HttpHost("myproxy", 8080))
        .bodyForm(Form.form().add("username", "vip").add("password", "secret").build())
        .execute().saveContent(new File("result.dump"));
```

One can also use Executor directly in order to execute requests in a specific security context whereby authentication details are cached and re-used for subsequent requests.

还可以直接使用 Executor 在特定的安全上下文中执行请求，从而缓存身份验证细节并为后续请求重用。

```
Executor executor = Executor.newInstance()
        .auth(new HttpHost("somehost"), "username", "password")
        .auth(new HttpHost("myproxy", 8080), "username", "password")
        .authPreemptive(new HttpHost("myproxy", 8080));

executor.execute(Request.Get("http://somehost/"))
        .returnContent().asString();

executor.execute(Request.Post("http://somehost/do-stuff")
        .useExpectContinue()
        .bodyString("Important stuff", ContentType.DEFAULT_TEXT))
        .returnContent().asString();
```

### 5.1.1. Response handling（响应处理）

The fluent facade API generally relieves the users from having to deal with connection management and resource deallocation. In most cases, though, this comes at a price of having to buffer content of response messages in memory. It is highly recommended to use ResponseHandler for HTTP response processing in order to avoid having to buffer content in memory.

外观模式 API 通常使用户不必处理连接管理和资源分配。但是，在大多数情况下，这样做的代价是必须在内存中缓冲响应消息的内容。强烈建议使用 ResponseHandler 进行 HTTP 响应处理，以避免在内存中缓冲内容。

```
Document result = Request.Get("http://somehost/content")
        .execute().handleResponse(new ResponseHandler<Document>() {

    public Document handleResponse(final HttpResponse response) throws IOException {
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
        DocumentBuilderFactory dbfac = DocumentBuilderFactory.newInstance();
        try {
            DocumentBuilder docBuilder = dbfac.newDocumentBuilder();
            ContentType contentType = ContentType.getOrDefault(entity);
            if (!contentType.equals(ContentType.APPLICATION_XML)) {
                throw new ClientProtocolException("Unexpected content type:" +
                    contentType);
            }
            String charset = contentType.getCharset();
            if (charset == null) {
                charset = HTTP.DEFAULT_CONTENT_CHARSET;
            }
            return docBuilder.parse(entity.getContent(), charset);
        } catch (ParserConfigurationException ex) {
            throw new IllegalStateException(ex);
        } catch (SAXException ex) {
            throw new ClientProtocolException("Malformed XML document", ex);
        }
    }

    });
```
