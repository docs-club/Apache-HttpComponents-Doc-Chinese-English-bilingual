## HttpClient Overview

The Hyper-Text Transfer Protocol (HTTP) is perhaps the most significant protocol used on the Internet today. Web services, network-enabled appliances and the growth of network computing continue to expand the role of the HTTP protocol beyond user-driven web browsers, while increasing the number of applications that require HTTP support.

超文本传输协议（HTTP）可能是当今 Internet 最重要的协议。Web 服务、支持网络的设备和网络计算的增长继续扩展了 HTTP 协议在用户驱动的 Web 浏览器之外的作用，同时增加了需要 HTTP 支持的应用程序的数量。`注：该译文在 Apache HttpComponents README、HttpClient Preface 第一段、HttpClient README 第一段、项目 README 均相同`

Although the java.net package provides basic functionality for accessing resources via HTTP, it doesn't provide the full flexibility or functionality needed by many applications. HttpClient seeks to fill this void by providing an efficient, up-to-date, and feature-rich package implementing the client side of the most recent HTTP standards and recommendations.

虽然 java.net 包提供了通过 HTTP 访问资源的基本功能，但它没有提供大多数应用程序所需的灵活性或功能。Jakarta Commons HttpClient 组件通过提供一个高效、崭新、功能丰富的包来实现最新 HTTP 标准的客户端，从而填补这一空白。有关标准兼容性和功能的更多细节，请参见特性页面。`注：该译文在 HttpClient Preface 第一段、HttpClient README 第一段、项目 README 均相同`

Designed for extension while providing robust support for the base HTTP protocol, HttpClient may be of interest to anyone building HTTP-aware client applications such as web browsers, web service clients, or systems that leverage or extend the HTTP protocol for distributed communication.

HttpClient 组件是为功能扩展而设计的，同时为基本 HTTP 协议提供强有力的支持，任何使用 HTTP 构建的客户端应用程序（如 web 浏览器、web 服务客户端或利用、扩展 HTTP 协议进行分布式通信的系统）的人都可能对它感兴趣。`注：该译文在 Apache HttpComponents README、HttpClient Preface 第一段、HttpClient README 第一段、项目 README 均相同`

## Documentation

1. Quick Start - contains a simple, complete example of an HTTP GET and POST with parameters.

[快速开始](/HttpClient/HttpClient-Quick-Start.md)。包含一个简单、带有参数的 HTTP GET 和 POST 的完整案例。

2. HttpClient Tutorial - gives a detailed examination of the HttpClient API, which was written in close accordance with the (sometimes not very intuitive) HTTP specification/standard. A copy is also shipped with the release. A PDF version is also available.

[HttpClient 教程](/HttpClient/HttpClient-Tutorial)，详细介绍了 HttpClient API，它是根据 HTTP 规范/标准编写的（有时不是很直观）。该版本还附带了一个副本。还有 PDF 版本。

3. HttpClient Examples - a set of examples demonstrating some of the more complex behavior.

[HttpClient 案例](/HttpClient/HttpClient-Examples.md)。通过一组示例演示一些更复杂的行为。

4. HttpClient Primer - explains the scope of HttpClient. Note that HttpClient is not a browser. It lacks the UI, HTML renderer and a JavaScript engine that a browser will possess.

[HttpClient 作用域](/HttpClient/Client-HTTP-Programming-Primer.md)。注意，HttpClient 不是一个浏览器。它缺少浏览器所拥有的 UI、HTML 渲染器和 JavaScript 引擎。

5. Project reports

- [HttpClient](http://hc.apache.org/httpcomponents-client-ga/httpclient/project-reports.html)
- [HC Fluent](http://hc.apache.org/httpcomponents-client-ga/fluent-hc/project-reports.html)
- [HttpMime](http://hc.apache.org/httpcomponents-client-ga/httpmime/project-reports.html)
- [HttpClient Cache](http://hc.apache.org/httpcomponents-client-ga/httpclient-cache/project-reports.html)
- [HttpClient OSGi](http://hc.apache.org/httpcomponents-client-ga/httpclient-osgi/project-reports.html)

## Features

- Standards based, pure Java, implementation of HTTP versions 1.0 and 1.1

基于标准，纯 Java，实现 HTTP 1.0 和 1.1

- Full implementation of all HTTP methods (GET, POST, PUT, DELETE, HEAD, OPTIONS, and TRACE) in an extensible OO framework.

可扩展的 OO 框架，全面实现所有 HTTP 方法（GET、POST、PUT、DELETE、HEAD、OPTIONS 和 TRACE）。

- Supports encryption with HTTPS (HTTP over SSL) protocol.

支持 HTTPS（HTTP over SSL）加密协议。

- Transparent connections through HTTP proxies.

透明 HTTP 连接。

- Tunneled HTTPS connections through HTTP proxies, via the CONNECT method.

通过 HTTP 代理，CONNECT 方法隧道 HTTPS 连接。

- Basic, Digest, NTLMv1, NTLMv2, NTLM2 Session, SNPNEGO, Kerberos authentication schemes.

Basic、Digest、NTLMv1、NTLMv2、NTLM2 会话，SNPNEGO、Kerberos 身份验证方案。

- Plug-in mechanism for custom authentication schemes.

基于插件机制的自定义认证方案。

- Pluggable secure socket factories, making it easier to use third party solutions

可插拔的安全插座工厂，使它更容易使用第三方解决方案

- Connection management support for use in multi-threaded applications. Supports setting the maximum total connections as well as the maximum connections per host. Detects and closes stale connections.

支持多线程应用的连接管理。支持设置最大总连接数以及每台主机的最大连接数。检测并关闭陈旧的连接。

- Automatic Cookie handling for reading Set-Cookie: headers from the server and sending them back out in a Cookie: header when appropriate.

- Plug-in mechanism for custom cookie policies.

自定义 cookie 策略的插件机制。

- Request output streams to avoid buffering any content body by streaming directly to the socket to the server.

请求输出流以避免通过直接流到服务器的套接字来缓冲任何内容主体。

- Response input streams to efficiently read the response body by streaming directly from the socket to the server.

响应输入流通过直接从套接字流到服务器来有效地读取响应主体。

- Persistent connections using KeepAlive in HTTP/1.0 and persistance in HTTP/1.1

在 HTTP/1.0 中使用 KeepAlive，在 HTTP/1.1 中使用持久性

- Direct access to the response code and headers sent by the server.

直接访问服务器发送的响应代码和消息头。

- The ability to set connection timeouts.

可设置连接超时。

- Support for HTTP/1.1 response caching.

支持 HTTP/1.1 响应缓存。

- Source code is freely available under the Apache License.

源代码可以在 Apache 许可下免费获得。

## Standards Compliance

HttpClient strives to conform to the following specifications endorsed by the Internet Engineering Task Force (IETF) and the internet at large:

HttpClient 努力遵守以下由互联网工程任务组（IETF）和整个互联网认可的行业规范：

- [RFC 1945](http://tools.ietf.org/html/rfc1945) Hypertext Transfer Protocol -- HTTP/1.0
- [RFC 2616](http://tools.ietf.org/html/rfc2616) Hypertext Transfer Protocol -- HTTP/1.1
- [RFC 2617](http://tools.ietf.org/html/rfc2617) HTTP Authentication: Basic and Digest Access Authentication
- [RFC 2396](http://tools.ietf.org/html/rfc2396) Uniform Resource Identifiers (URI): Generic Syntax
- [RFC 6265](http://tools.ietf.org/html/rfc6265) HTTP State Management Mechanism (Cookies)
