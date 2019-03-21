## Apache HttpComponents
The Apache HttpComponents™ project is responsible for creating and maintaining a toolset of low level Java components focused on HTTP and associated protocols.

This project functions under the Apache Software Foundation (http://www.apache.org), and is part of a larger community of developers and users.

## HttpComponents Overview
The Hyper-Text Transfer Protocol (HTTP) is perhaps the most significant protocol used on the Internet today. Web services, network-enabled appliances and the growth of network computing continue to expand the role of the HTTP protocol beyond user-driven web browsers, while increasing the number of applications that require HTTP support.

Designed for extension while providing robust support for the base HTTP protocol, the HttpComponents may be of interest to anyone building HTTP-aware client and server applications such as web browsers, web spiders, HTTP proxies, web service transport libraries, or systems that leverage or extend the HTTP protocol for distributed communication.

## HttpComponents Structure

### HttpComponents Core
HttpCore is a set of low level HTTP transport components that can be used to build custom client and server side HTTP services with a minimal footprint. HttpCore supports two I/O models: blocking I/O model based on the classic Java I/O and non-blocking, event driven I/O model based on Java NIO.

The blocking I/O model may be more appropriate for data intensive, low latency scenarios, whereas the non-blocking model may be more appropriate for high latency scenarios where raw data throughput is less important than the ability to handle thousands of simultaneous HTTP connections in a resource efficient manner.

- HttpCore Tutorial HTML / PDF
- HttpCore Examples

### HttpComponents Client
HttpClient is a HTTP/1.1 compliant HTTP agent implementation based on HttpCore. It also provides reusable components for client-side authentication, HTTP state management, and HTTP connection management. HttpComponents Client is a successor of and replacement for Commons HttpClient 3.x. Users of Commons HttpClient are strongly encouraged to upgrade.

- HttpClient Tutorial HTML / PDF
- HttpClient Samples
- HttpClient port for Android

### HttpComponents AsyncClient
Asynch HttpClient is a HTTP/1.1 compliant HTTP agent implementation based on HttpCore NIO and HttpClient components. It is a complementary module to Apache HttpClient intended for special cases where ability to handle a great number of concurrent connections is more important than performance in terms of a raw data throughput.

- HttpAsyncClient Samples

### Commons HttpClient (legacy)
Commons HttpClient 3.x codeline is at the end of life. All users of Commons HttpClient 3.x are strongly encouraged to upgrade to HttpClient 4.1.
