# Chapter 7. Advanced topics

## 7.1. Custom client connections

In certain situations it may be necessary to customize the way HTTP messages get transmitted across the wire beyond what is possible using HTTP parameters in order to be able to deal non-standard, non-compliant behaviours. For instance, for web crawlers it may be necessary to force HttpClient into accepting malformed response heads in order to salvage the content of the messages.

在某些情况下，为了能够处理非标准的、不兼容的行为，可能需要定制 HTTP 消息通过网络传输的方式，而不是使用 HTTP 参数。例如，对于 web 爬虫程序，可能需要强制 HttpClient 接受格式不正确的响应头，以便回收消息的内容。

Usually the process of plugging in a custom message parser or a custom connection implementation involves several steps:

通常，插入自定义消息解析器或自定义连接实现的过程包括几个步骤：

- Provide a custom LineParser / LineFormatter interface implementation. Implement message parsing / formatting logic as required.

提供自定义 LineParser/LineFormatter 接口实现。根据需要实现消息解析/格式化逻辑。

```
class MyLineParser extends BasicLineParser {

    @Override
    public Header parseHeader(
            CharArrayBuffer buffer) throws ParseException {
        try {
            return super.parseHeader(buffer);
        } catch (ParseException ex) {
            // Suppress ParseException exception
            return new BasicHeader(buffer.toString(), null);
        }
    }

}
```

- Provide a custom HttpConnectionFactory implementation. Replace default request writer and / or response parser with custom ones as required.

提供一个自定义的 HttpConnectionFactory 实现。根据需要用定制的请求编写器和/或响应解析器替换默认的请求编写器和/或响应解析器。

```
HttpConnectionFactory<HttpRoute, ManagedHttpClientConnection> connFactory =
        new ManagedHttpClientConnectionFactory(
            new DefaultHttpRequestWriterFactory(),
            new DefaultHttpResponseParserFactory(
                    new MyLineParser(), new DefaultHttpResponseFactory()));
```

- Configure HttpClient to use the custom connection factory.

配置 HttpClient 以使用自定义连接工厂。

```
PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager(
    connFactory);
CloseableHttpClient httpclient = HttpClients.custom()
        .setConnectionManager(cm)
        .build();
```

## 7.2. Stateful HTTP connections

While HTTP specification assumes that session state information is always embedded in HTTP messages in the form of HTTP cookies and therefore HTTP connections are always stateless, this assumption does not always hold true in real life. There are cases when HTTP connections are created with a particular user identity or within a particular security context and therefore cannot be shared with other users and can be reused by the same user only. Examples of such stateful HTTP connections are NTLM authenticated connections and SSL connections with client certificate authentication.

虽然 HTTP 规范假设会话状态信息总是以 HTTP cookie 的形式嵌入到 HTTP 消息中，因此 HTTP 连接总是无状态的，但这种假设在实际生活中并不总是成立。在某些情况下，HTTP 连接是使用特定的用户标识或在特定的安全上下文中创建的，因此不能与其他用户共享，只能由同一用户重用。这种有状态 HTTP 连接的例子有 NTLM 认证连接和客户端证书认证的 SSL 连接。

### 7.2.1. User token handler

HttpClient relies on UserTokenHandler interface to determine if the given execution context is user specific or not. The token object returned by this handler is expected to uniquely identify the current user if the context is user specific or to be null if the context does not contain any resources or details specific to the current user. The user token will be used to ensure that user specific resources will not be shared with or reused by other users.

HttpClient 依赖 UserTokenHandler 接口来确定给定的执行上下文是否特定于用户。如果上下文是特定于用户的，则此处理程序返回的令牌对象将惟一标识当前用户；如果上下文不包含特定于当前用户的任何资源或详细信息，则该令牌对象将为 null。用户令牌将用于确保不会与其他用户共享或重用用户特定的资源。

The default implementation of the UserTokenHandler interface uses an instance of Principal class to represent a state object for HTTP connections, if it can be obtained from the given execution context. DefaultUserTokenHandler will use the user principal of connection based authentication schemes such as NTLM or that of the SSL session with client authentication turned on. If both are unavailable, null token will be returned.

UserTokenHandler 接口的默认实现使用 Principal 类的一个实例来表示 HTTP 连接的状态对象（如果可以从给定的执行上下文获得）。DefaultUserTokenHandler 将使用基于连接的身份验证方案的用户主体，如 NTLM 或启用了客户端身份验证的 SSL 会话的用户主体。如果两者都不可用，则返回 null 令牌。

```
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpClientContext context = HttpClientContext.create();
HttpGet httpget = new HttpGet("http://localhost:8080/");
CloseableHttpResponse response = httpclient.execute(httpget, context);
try {
    Principal principal = context.getUserToken(Principal.class);
    System.out.println(principal);
} finally {
    response.close();
}
```

Users can provide a custom implementation if the default one does not satisfy their needs:

用户可以提供一个自定义实现，如果默认的不满足他们的需要：

```
UserTokenHandler userTokenHandler = new UserTokenHandler() {

    public Object getUserToken(HttpContext context) {
        return context.getAttribute("my-token");
    }

};
CloseableHttpClient httpclient = HttpClients.custom()
        .setUserTokenHandler(userTokenHandler)
        .build();
```

### 7.2.2. Persistent stateful connections

Please note that a persistent connection that carries a state object can be reused only if the same state object is bound to the execution context when requests are executed. So, it is really important to ensure the either same context is reused for execution of subsequent HTTP requests by the same user or the user token is bound to the context prior to request execution.

请注意，只有在执行请求时将相同的状态对象绑定到执行上下文，才能重用带有状态对象的持久连接。因此，确保同一个用户在执行后续 HTTP 请求时重用相同的上下文，或者在执行请求之前将用户令牌绑定到上下文，这一点非常重要。

```
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpClientContext context1 = HttpClientContext.create();
HttpGet httpget1 = new HttpGet("http://localhost:8080/");
CloseableHttpResponse response1 = httpclient.execute(httpget1, context1);
try {
    HttpEntity entity1 = response1.getEntity();
} finally {
    response1.close();
}
Principal principal = context1.getUserToken(Principal.class);

HttpClientContext context2 = HttpClientContext.create();
context2.setUserToken(principal);
HttpGet httpget2 = new HttpGet("http://localhost:8080/");
CloseableHttpResponse response2 = httpclient.execute(httpget2, context2);
try {
    HttpEntity entity2 = response2.getEntity();
} finally {
    response2.close();
}
```

## 7.3. Using the FutureRequestExecutionService

Using the FutureRequestExecutionService, you can schedule http calls and treat the response as a Future. This is useful when e.g. making multiple calls to a web service. The advantage of using the FutureRequestExecutionService is that you can use multiple threads to schedule requests concurrently, set timeouts on the tasks, or cancel them when a response is no longer necessary.

使用 FutureRequestExecutionService，您可以调度 http 调用并将响应作为未来处理。这在对 web 服务进行多次调用时非常有用。使用 FutureRequestExecutionService 的优点是，您可以使用多个线程来同时调度请求，设置任务的超时，或者在不再需要响应时取消请求。

FutureRequestExecutionService wraps the request with a HttpRequestFutureTask, which extends FutureTask. This class allows you to cancel the task as well as keep track of various metrics such as request duration.

FutureRequestExecutionService 使用 HttpRequestFutureTask 封装请求，该请求扩展了 FutureTask。这个类允许您取消任务，并跟踪各种指标，如请求持续时间。

### 7.3.1. Creating the FutureRequestExecutionService

The constructor for the futureRequestExecutionService takes any existing httpClient instance and an ExecutorService instance. When configuring both, it is important to align the maximum number of connections with the number of threads you are going to use. When there are more threads than connections, the connections may start timing out because there are no available connections. When there are more connections than threads, the futureRequestExecutionService will not use all of them.

futureRequestExecutionService 的构造函数接受任何现有的 httpClient 实例和 ExecutorService 实例。在配置这两个线程时，将最大连接数与将要使用的线程数对齐非常重要。当线程多于连接时，连接可能开始超时，因为没有可用的连接。当连接数多于线程数时，未来的马术执行服务将不会全部使用它们。

```
HttpClient httpClient = HttpClientBuilder.create().setMaxConnPerRoute(5).build();
ExecutorService executorService = Executors.newFixedThreadPool(5);
FutureRequestExecutionService futureRequestExecutionService =
    new FutureRequestExecutionService(httpClient, executorService);
```

### 7.3.2. Scheduling requests

To schedule a request, simply provide a HttpUriRequest, HttpContext, and a ResponseHandler. Because the request is processed by the executor service, a ResponseHandler is mandatory.

要调度一个请求，只需提供一个 HttpUriRequest、HttpContext 和一个 ResponseHandler。因为请求是由 executor 服务处理的，所以必须使用 ResponseHandler。

```
private final class OkidokiHandler implements ResponseHandler<Boolean> {
    public Boolean handleResponse(
            final HttpResponse response) throws ClientProtocolException, IOException {
        return response.getStatusLine().getStatusCode() == 200;
    }
}

HttpRequestFutureTask<Boolean> task = futureRequestExecutionService.execute(
    new HttpGet("http://www.google.com"), HttpClientContext.create(),
    new OkidokiHandler());
// blocks until the request complete and then returns true if you can connect to Google
boolean ok=task.get();
```

### 7.3.3. Canceling tasks

Scheduled tasks may be cancelled. If the task is not yet executing but merely queued for execution, it simply will never execute. If it is executing and the mayInterruptIfRunning parameter is set to true, abort() will be called on the request; otherwise the response will simply be ignored but the request will be allowed to complete normally. Any subsequent calls to task.get() will fail with an IllegalStateException. It should be noticed that canceling tasks merely frees up the client side resources. The request may actually be handled normally on the server side.

预定的任务可能会被取消。如果任务还没有执行，只是排队等待执行，那么它将永远不会执行。如果它正在执行，并且 mayInterruptIfRunning 参数被设置为 true，则将对请求调用 abort()；否则，响应将被简单地忽略，但请求将被允许正常完成。任何对 task.get() 的后续调用都将失败，并带有一个 IllegalStateException。应该注意，取消任务只会释放客户端资源。请求实际上可以在服务器端正常处理。

```
task.cancel(true)
task.get() // throws an Exception
```

### 7.3.4. Callbacks

Instead of manually calling task.get(), you can also use a FutureCallback instance that gets callbacks when the request completes. This is the same interface as is used in HttpAsyncClient

除了手动调用 task.get() 之外，您还可以使用 FutureCallback 实例，该实例在请求完成时获取回调。这个接口与 HttpAsyncClient 中使用的接口相同。

```
private final class MyCallback implements FutureCallback<Boolean> {

    public void failed(final Exception ex) {
        // do something
    }

    public void completed(final Boolean result) {
        // do something
    }

    public void cancelled() {
        // do something
    }
}

HttpRequestFutureTask<Boolean> task = futureRequestExecutionService.execute(
    new HttpGet("http://www.google.com"), HttpClientContext.create(),
    new OkidokiHandler(), new MyCallback());
```

### 7.3.5. Metrics

FutureRequestExecutionService is typically used in applications that make large amounts of web service calls. To facilitate e.g. monitoring or configuration tuning, the FutureRequestExecutionService keeps track of several metrics.

FutureRequestExecutionService 通常用于执行大量 web 服务调用的应用程序中。为了方便监视或配置调优，FutureRequestExecutionService 会跟踪一些指标。

Each HttpRequestFutureTask provides methods to get the time the task was scheduled, started, and ended. Additionally, request and task duration are available as well. These metrics are aggregated in the FutureRequestExecutionService in a FutureRequestExecutionMetrics instance that may be accessed through FutureRequestExecutionService.metrics().

每个 HttpRequestFutureTask 都提供一些方法来获取任务调度、启动和结束的时间。此外，还可以使用请求和任务持续时间。指标聚合在 FutureRequestExecutionService 中的 FutureRequestExecutionMetrics 实例中，该实例可以通过 FutureRequestExecutionService.metrics() 访问。

```
task.scheduledTime() // returns the timestamp the task was scheduled
task.startedTime() // returns the timestamp when the task was started
task.endedTime() // returns the timestamp when the task was done executing
task.requestDuration // returns the duration of the http request
task.taskDuration // returns the duration of the task from the moment it was scheduled

FutureRequestExecutionMetrics metrics = futureRequestExecutionService.metrics()
metrics.getActiveConnectionCount() // currently active connections
metrics.getScheduledConnectionCount(); // currently scheduled connections
metrics.getSuccessfulConnectionCount(); // total number of successful requests
metrics.getSuccessfulConnectionAverageDuration(); // average request duration
metrics.getFailedConnectionCount(); // total number of failed tasks
metrics.getFailedConnectionAverageDuration(); // average duration of failed tasks
metrics.getTaskCount(); // total number of tasks scheduled
metrics.getRequestCount(); // total number of requests
metrics.getRequestAverageDuration(); // average request duration
metrics.getTaskAverageDuration(); // average task duration
```
