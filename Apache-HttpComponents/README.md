## Apache HttpComponents

The Apache HttpComponents™ project is responsible for creating and maintaining a toolset of low level Java components focused on HTTP and associated protocols.

Apache HttpComponents™ 项目负责创建和维护一套底层 Java 组件工具集，这些组件主要关注 HTTP 和相关协议。

This project functions under the Apache Software Foundation (http://www.apache.org), and is part of a larger community of developers and users.

这个项目在 Apache Software Foundation（http://www.apache.org）下运行，它是一个大型开发人员和用户社区的一部分。

## HttpComponents Overview（HttpComponents 概述）

The Hyper-Text Transfer Protocol (HTTP) is perhaps the most significant protocol used on the Internet today. Web services, network-enabled appliances and the growth of network computing continue to expand the role of the HTTP protocol beyond user-driven web browsers, while increasing the number of applications that require HTTP support.

超文本传输协议（HTTP）可能是当今 Internet 最重要的协议。Web 服务、支持网络的设备和网络计算的增长继续扩展了 HTTP 协议在用户驱动的 Web 浏览器之外的作用，同时增加了需要 HTTP 支持的应用程序的数量。

Designed for extension while providing robust support for the base HTTP protocol, the HttpComponents may be of interest to anyone building HTTP-aware client and server applications such as web browsers, web spiders, HTTP proxies, web service transport libraries, or systems that leverage or extend the HTTP protocol for distributed communication.

HttpClient 组件是为功能扩展而设计的，同时为基本 HTTP 协议提供强有力的支持，任何使用 HTTP 构建的客户端应用程序（如 web 浏览器、web 服务客户端或利用、扩展 HTTP 协议进行分布式通信的系统）的人都可能对它感兴趣。

## HttpComponents Structure（HttpComponents 的构成）

### HttpComponents Core（HttpComponents 核心）

HttpCore is a set of low level HTTP transport components that can be used to build custom client and server side HTTP services with a minimal footprint. HttpCore supports two I/O models: blocking I/O model based on the classic Java I/O and non-blocking, event driven I/O model based on Java NIO.

HttpCore 是一组底层 HTTP 传输组件，可用于构建自定义客户端和服务器端 HTTP 服务，并且占用空间最少。HttpCore 支持两种 I/O 模型：基于经典 Java I/O 的阻塞 I/O 模型和基于 Java NIO 的非阻塞事件驱动 I/O 模型。

The blocking I/O model may be more appropriate for data intensive, low latency scenarios, whereas the non-blocking model may be more appropriate for high latency scenarios where raw data throughput is less important than the ability to handle thousands of simultaneous HTTP connections in a resource efficient manner.

阻塞 I/O 模型可能更适合于数据密集型、低延迟的场景，而非阻塞模型可能更适合于高延迟的场景，在这些场景中，原始数据吞吐量不如以资源有效的方式处理数千个并发 HTTP 连接的能力重要。

- HttpCore Tutorial HTML / PDF

HttpCore 教程

- HttpCore Examples

HttpCore 案例

### HttpComponents Client（HttpComponents 客户端）

HttpClient is a HTTP/1.1 compliant HTTP agent implementation based on HttpCore. It also provides reusable components for client-side authentication, HTTP state management, and HTTP connection management. HttpComponents Client is a successor of and replacement for Commons HttpClient 3.x. Users of Commons HttpClient are strongly encouraged to upgrade.

HttpClient 是一个基于 HttpCore 并与 HTTP/1.1 兼容的 HTTP 代理实现。它还为客户端身份验证、HTTP 状态管理和 HTTP 连接管理提供了可重用组件。HttpComponents Client 是 Commons HttpClient 3.x 的继承者和替代者。强烈建议使用 Commons HttpClient 的用户进行升级。

- HttpClient Tutorial HTML / PDF

HttpClient 教程

- HttpClient Samples

HttpClient 案例

- HttpClient port for Android

安卓下的 HttpClient 接口

### HttpComponents AsyncClient（HttpComponents 的异步客户端）

Asynch HttpClient is a HTTP/1.1 compliant HTTP agent implementation based on HttpCore NIO and HttpClient components. It is a complementary module to Apache HttpClient intended for special cases where ability to handle a great number of concurrent connections is more important than performance in terms of a raw data throughput.

异步 HttpClient 是一个基于 HttpCore NIO 和 HttpClient 组件的 HTTP/1.1 兼容的 HTTP 代理实现。它是 Apache HttpClient 的一个补充模块，适用于处理大量并发连接的能力比处理原始数据吞吐量的性能更重要的特殊情况。

- HttpAsyncClient Samples

HttpAsyncClient 案例

### Commons HttpClient (legacy)

Commons HttpClient 3.x codeline is at the end of life. All users of Commons HttpClient 3.x are strongly encouraged to upgrade to HttpClient 4.1.

Commons HttpClient 3.x 代码的生命已终结。强烈建议所有 Commons HttpClient 3.x 用户升级到 HttpClient 4.1。