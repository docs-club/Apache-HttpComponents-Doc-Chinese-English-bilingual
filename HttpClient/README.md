## HttpClient Overview
The Hyper-Text Transfer Protocol (HTTP) is perhaps the most significant protocol used on the Internet today. Web services, network-enabled appliances and the growth of network computing continue to expand the role of the HTTP protocol beyond user-driven web browsers, while increasing the number of applications that require HTTP support.

Although the java.net package provides basic functionality for accessing resources via HTTP, it doesn't provide the full flexibility or functionality needed by many applications. HttpClient seeks to fill this void by providing an efficient, up-to-date, and feature-rich package implementing the client side of the most recent HTTP standards and recommendations.

Designed for extension while providing robust support for the base HTTP protocol, HttpClient may be of interest to anyone building HTTP-aware client applications such as web browsers, web service clients, or systems that leverage or extend the HTTP protocol for distributed communication.

## Documentation
1. Quick Start - contains a simple, complete example of an HTTP GET and POST with parameters.

2. HttpClient Tutorial - gives a detailed examination of the HttpClient API, which was written in close accordance with the (sometimes not very intuitive) HTTP specification/standard. A copy is also shipped with the release. A PDF version is also available

3. HttpClient Examples - a set of examples demonstrating some of the more complex behavior.

4. HttpClient Primer - explains the scope of HttpClient. Note that HttpClient is not a browser. It lacks the UI, HTML renderer and a JavaScript engine that a browser will possess.

5. Project reports
    - HttpClient
    - HC Fluent
    - HttpMime
    - HttpClient Cache
    - HttpClient OSGi

## Features
- Standards based, pure Java, implementation of HTTP versions 1.0 and 1.1
- Full implementation of all HTTP methods (GET, POST, PUT, DELETE, HEAD, OPTIONS, and TRACE) in an extensible OO framework.
- Supports encryption with HTTPS (HTTP over SSL) protocol.
- Transparent connections through HTTP proxies.
- Tunneled HTTPS connections through HTTP proxies, via the CONNECT method.
- Basic, Digest, NTLMv1, NTLMv2, NTLM2 Session, SNPNEGO, Kerberos authentication schemes.
- Plug-in mechanism for custom authentication schemes.
- Pluggable secure socket factories, making it easier to use third party solutions
- Connection management support for use in multi-threaded applications. Supports setting the maximum total connections as well as the maximum connections per host. Detects and closes stale connections.
- Automatic Cookie handling for reading Set-Cookie: headers from the server and sending them back out in a Cookie: header when appropriate.
- Plug-in mechanism for custom cookie policies.
- Request output streams to avoid buffering any content body by streaming directly to the socket to the server.
- Response input streams to efficiently read the response body by streaming directly from the socket to the server.
- Persistent connections using KeepAlive in HTTP/1.0 and persistance in HTTP/1.1
- Direct access to the response code and headers sent by the server.
- The ability to set connection timeouts.
- Support for HTTP/1.1 response caching.
- Source code is freely available under the Apache License.

## Standards Compliance
HttpClient strives to conform to the following specifications endorsed by the Internet Engineering Task Force (IETF) and the internet at large:

- RFC 1945 Hypertext Transfer Protocol -- HTTP/1.0
- RFC 2616 Hypertext Transfer Protocol -- HTTP/1.1
- RFC 2617 HTTP Authentication: Basic and Digest Access Authentication
- RFC 6265 HTTP State Management Mechanism (Cookies)
