## HttpClient Examples

- [Response handling](http://hc.apache.org/httpcomponents-client-ga/httpclient/examples/org/apache/http/examples/client/ClientWithResponseHandler.java)

This example demonstrates how to process HTTP responses using a response handler. This is the recommended way of executing HTTP requests and processing HTTP responses. This approach enables the caller to concentrate on the process of digesting HTTP responses and to delegate the task of system resource deallocation to HttpClient. The use of an HTTP response handler guarantees that the underlying HTTP connection will be released back to the connection manager automatically in all cases.

此示例演示如何使用响应处理程序处理 HTTP 响应。这是执行 HTTP 请求和处理 HTTP 响应的推荐方法。这种方法使调用者能够集中精力处理 HTTP 响应，并将系统资源分配的任务委托给 HttpClient。HTTP 响应处理程序的使用保证了底层 HTTP 连接在所有情况下自动释放回连接管理器。

- [Manual connection release](http://hc.apache.org/httpcomponents-client-ga/httpclient/examples/org/apache/http/examples/client/ClientConnectionRelease.java)

This example demonstrates how to ensure the release of the underlying HTTP connection back to the connection manager in case of a manual processing of HTTP responses.

这个示例演示了如何确保在手动处理 HTTP 响应时将底层 HTTP 连接释放回连接管理器。

- [HttpClient configuration](http://hc.apache.org/httpcomponents-client-ga/httpclient/examples/org/apache/http/examples/client/ClientConfiguration.java)

This example demonstrates how to customize and configure the most common aspects of HTTP request execution and connection management.

这个案例展示了如何自定义和配置最常见的 HTTP 请求执行和连接管理的各个方面。

- [Abort method](http://hc.apache.org/httpcomponents-client-ga/httpclient/examples/org/apache/http/examples/client/ClientAbortMethod.java)

This example demonstrates how to abort an HTTP request before its normal completion.

这个例子演示了如何在 HTTP 请求正常完成之前中止它。

- [Client authentication](http://hc.apache.org/httpcomponents-client-ga/httpclient/examples/org/apache/http/examples/client/ClientAuthentication.java)

This example uses HttpClient to execute an HTTP request against a target site that requires user authentication.

此示例使用 HttpClient 对需要用户身份验证的目标站点执行 HTTP 请求。

- [Request via a proxy](http://hc.apache.org/httpcomponents-client-ga/httpclient/examples/org/apache/http/examples/client/ClientExecuteProxy.java)

This example demonstrates how to send an HTTP request via a proxy.

这个示例演示了如何通过代理发送 HTTP 请求。

- [Proxy authentication](http://hc.apache.org/httpcomponents-client-ga/httpclient/examples/org/apache/http/examples/client/ClientProxyAuthentication.java)

A simple example showing execution of an HTTP request over a secure connection tunneled through an authenticating proxy.

这是一个简单的示例，演示了通过身份验证代理在安全连接隧道上执行 HTTP 请求。

- [Chunk encoded POST](http://hc.apache.org/httpcomponents-client-ga/httpclient/examples/org/apache/http/examples/client/ClientChunkEncodedPost.java)

This example shows how to stream out a request entity using chunk encoding.

这个示例展示了如何使用 chunk 编码来发送请求实体。

- [Custom execution context](http://hc.apache.org/httpcomponents-client-ga/httpclient/examples/org/apache/http/examples/client/ClientCustomContext.java)

This example demonstrates the use of a local HTTP context populated custom attributes.

的示例演示了使用本地 HTTP 上下文填充的自定义属性。

- [Form based logon](http://hc.apache.org/httpcomponents-client-ga/httpclient/examples/org/apache/http/examples/client/ClientFormLogin.java)

This example demonstrates how HttpClient can be used to perform form-based logon.

这个示例演示了如何使用 HttpClient 执行基于表单的登录。

- [Threaded request execution](http://hc.apache.org/httpcomponents-client-ga/httpclient/examples/org/apache/http/examples/client/ClientMultiThreadedExecution.java)

An example that executes HTTP requests from multiple worker threads.

一个多线程执行 HTTP 请求的例子。

- [Custom SSL context](http://hc.apache.org/httpcomponents-client-ga/httpclient/examples/org/apache/http/examples/client/ClientCustomSSL.java)

  This example demonstrates how to create secure connections with a custom SSL context.

此示例演示如何使用自定义 SSL 上下文创建安全连接。

- [Preemptive BASIC authentication](http://hc.apache.org/httpcomponents-client-ga/httpclient/examples/org/apache/http/examples/client/ClientPreemptiveBasicAuthentication.java)

This example shows how HttpClient can be customized to authenticate preemptively using BASIC scheme. Generally, preemptive authentication can be considered less secure than a response to an authentication challenge and therefore discouraged.

- [Preemptive DIGEST authentication](http://hc.apache.org/httpcomponents-client-ga/httpclient/examples/org/apache/http/examples/client/ClientPreemptiveDigestAuthentication.java)

This example shows how HttpClient can be customized to authenticate preemptively using DIGEST scheme. Generally, preemptive authentication can be considered less secure than a response to an authentication challenge and therefore discouraged.

- [Proxy tunnel](http://hc.apache.org/httpcomponents-client-ga/httpclient/examples/org/apache/http/examples/client/ProxyTunnelDemo.java)

This example shows how to use ProxyClient in order to establish a tunnel through an HTTP proxy for an arbitrary protocol.

这个示例演示了如何使用 ProxyClient 通过 HTTP 代理为任意协议建立隧道。

- [Multipart encoded request entity](http://hc.apache.org/httpcomponents-client-ga/httpmime/examples/org/apache/http/examples/entity/mime/ClientMultipartFormPost.java)

This example shows how to execute requests enclosing a multipart encoded entity.

此示例演示如何执行包含多部分编码实体的请求。

- [Native Windows Negotiate/NTLM](http://hc.apache.org/httpcomponents-client-ga/httpclient-win/examples/org/apache/http/examples/client/win/ClientWinAuth.java)

This example shows how to make use of Native Windows Negotiate/NTLM authentication when running on Windows OS.

这个示例演示了如何在 Windows 操作系统上运行时使用本机 Windows Negotiate/NTLM 身份验证。
