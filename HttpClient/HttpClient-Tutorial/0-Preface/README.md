# Preface

The Hyper-Text Transfer Protocol (HTTP) is perhaps the most significant protocol used on the Internet today. Web services, network-enabled appliances and the growth of network computing continue to expand the role of the HTTP protocol beyond user-driven web browsers, while increasing the number of applications that require HTTP support.

超文本传输协议（HTTP）可能是当今 Internet 最重要的协议。Web 服务、支持网络的设备和网络计算的增长继续扩展了 HTTP 协议在用户驱动的 Web 浏览器之外的作用，同时增加了需要 HTTP 支持的应用程序的数量。`注：该译文在 Apache HttpComponents README、HttpClient Preface 第一段、HttpClient README 第一段、项目 README 均相同`

Although the java.net package provides basic functionality for accessing resources via HTTP, it doesn't provide the full flexibility or functionality needed by many applications. HttpClient seeks to fill this void by providing an efficient, up-to-date, and feature-rich package implementing the client side of the most recent HTTP standards and recommendations.

虽然 java.net 包提供了通过 HTTP 访问资源的基本功能，但它没有提供大多数应用程序所需的灵活性或功能。Jakarta Commons HttpClient 组件通过提供一个高效、崭新、功能丰富的包来实现最新 HTTP 标准的客户端，从而填补这一空白。有关标准兼容性和功能的更多细节，请参见特性页面。`注：该译文在 HttpClient Preface 第一段、HttpClient README 第一段、项目 README 均相同`

Designed for extension while providing robust support for the base HTTP protocol, HttpClient may be of interest to anyone building HTTP-aware client applications such as web browsers, web service clients, or systems that leverage or extend the HTTP protocol for distributed communication.

HttpClient 组件是为功能扩展而设计的，同时为基本 HTTP 协议提供强有力的支持，任何使用 HTTP 构建的客户端应用程序（如 web 浏览器、web 服务客户端或利用、扩展 HTTP 协议进行分布式通信的系统）的人都可能对它感兴趣。`注：该译文在 Apache HttpComponents README、HttpClient Preface 第一段、HttpClient README 第一段、项目 README 均相同`

## 1. HttpClient scope

Client-side HTTP transport library based on HttpCore

基于 [HttpCore](https://github.com/clxering/Apache-HttpComponents-Doc-Chinese-English-bilingual/tree/master/HttpCore) 的客户端 HTTP 传输库

Based on classic (blocking) I/O

基于经典的（阻塞）I/O 模型

Content agnostic

与内容无关

## 2. What HttpClient is NOT

HttpClient is NOT a browser. It is a client side HTTP transport library. HttpClient's purpose is to transmit and receive HTTP messages. HttpClient will not attempt to process content, execute javascript embedded in HTML pages, try to guess content type, if not explicitly set, or reformat request / rewrite location URIs, or other functionality unrelated to the HTTP transport.

HttpClient 不是一个浏览器。它是一个客户端 HTTP 传输库。HttpClient 的目的是传输和接收 HTTP 消息。HttpClient 不会试图处理内容、执行嵌入在 HTML 页面中的 javascript、尝试猜测内容类型（如果没有显式设置）、or reformat request / rewrite location URIs 或其他与 HTTP 传输无关的功能。

---

**[Back to contents of HttpClient Tutorial（返回 HttpClient 教程目录）](https://github.com/clxering/Apache-HttpComponents-Doc-Chinese-English-bilingual/tree/master/HttpClient/HttpClient-Tutorial#contents)**

- **Next Chapter：[Chapter 1 Fundamentals](https://github.com/clxering/Apache-HttpComponents-Doc-Chinese-English-bilingual/tree/master/HttpClient/HttpClient-Tutorial/1-Fundamentals#chapter-1-fundamentals)**
