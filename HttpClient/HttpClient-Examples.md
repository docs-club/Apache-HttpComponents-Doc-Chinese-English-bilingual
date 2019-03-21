## HttpClient Examples

- Response handling
This example demonstrates how to process HTTP responses using a response handler. This is the recommended way of executing HTTP requests and processing HTTP responses. This approach enables the caller to concentrate on the process of digesting HTTP responses and to delegate the task of system resource deallocation to HttpClient. The use of an HTTP response handler guarantees that the underlying HTTP connection will be released back to the connection manager automatically in all cases.

- Manual connection release
This example demonstrates how to ensure the release of the underlying HTTP connection back to the connection manager in case of a manual processing of HTTP responses.

- HttpClient configuration
This example demonstrates how to customize and configure the most common aspects of HTTP request execution and connection management.

- Abort method
This example demonstrates how to abort an HTTP request before its normal completion.

- Client authentication
This example uses HttpClient to execute an HTTP request against a target site that requires user authentication.

- Request via a proxy
This example demonstrates how to send an HTTP request via a proxy.

- Proxy authentication
A simple example showing execution of an HTTP request over a secure connection tunneled through an authenticating proxy.

- Chunk encoded POST
This example shows how to stream out a request entity using chunk encoding.

- Custom execution context
This example demonstrates the use of a local HTTP context populated custom attributes.

- Form based logon
This example demonstrates how HttpClient can be used to perform form-based logon.

- Threaded request execution
An example that executes HTTP requests from multiple worker threads.

- Custom SSL context
This example demonstrates how to create secure connections with a custom SSL context.

- Preemptive BASIC authentication
This example shows how HttpClient can be customized to authenticate preemptively using BASIC scheme. Generally, preemptive authentication can be considered less secure than a response to an authentication challenge and therefore discouraged.

- Preemptive DIGEST authentication
This example shows how HttpClient can be customized to authenticate preemptively using DIGEST scheme. Generally, preemptive authentication can be considered less secure than a response to an authentication challenge and therefore discouraged.

- Proxy tunnel
This example shows how to use ProxyClient in order to establish a tunnel through an HTTP proxy for an arbitrary protocol.

- Multipart encoded request entity
This example shows how to execute requests enclosing a multipart encoded entity.

- Native Windows Negotiate/NTLM
This example shows how to make use of Native Windows Negotiate/NTLM authentication when running on Windows OS.
