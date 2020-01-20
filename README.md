# Commons-HttpClient-Doc-Chinese-English-bilingual

link to：http://hc.apache.org/httpclient-3.x/

## End of life（生命结束）

The Commons HttpClient project is now end of life, and is no longer being developed. It has been replaced by the Apache HttpComponents project in its HttpClient and HttpCore modules, which offer better performance and more flexibility.

Commons HttpClient 项目已经终结了，并且不再开发。它已经被 Apache HttpComponents 项目中的 HttpClient 和 HttpCore 模块取代，这些模块提供了更好的性能和更强的灵活性。

## Introduction（介绍）

The Hyper-Text Transfer Protocol (HTTP) is perhaps the most significant protocol used on the Internet today. Web services, network-enabled appliances and the growth of network computing continue to expand the role of the HTTP protocol beyond user-driven web browsers, while increasing the number of applications that require HTTP support.

超文本传输协议（HTTP）可能是当今 Internet 最重要的协议。Web 服务、支持网络的设备和网络计算的增长继续扩展了 HTTP 协议在用户驱动的 Web 浏览器之外的作用，同时增加了需要 HTTP 支持的应用程序的数量。

Although the java.net package provides basic functionality for accessing resources via HTTP, it doesn't provide the full flexibility or functionality needed by many applications. The Jakarta Commons HttpClient component seeks to fill this void by providing an efficient, up-to-date, and feature-rich package implementing the client side of the most recent HTTP standards and recommendations. See the Features page for more details on standards compliance and capabilities.

虽然 java.net 包提供了通过 HTTP 访问资源的基本功能，但它没有提供大多数应用程序所需的灵活性或功能。Jakarta Commons HttpClient 组件通过提供一个高效、崭新、功能丰富的包来实现最新 HTTP 标准的客户端，从而填补这一空白。有关标准兼容性和功能的更多细节，请参见特性页面。

Designed for extension while providing robust support for the base HTTP protocol, the HttpClient component may be of interest to anyone building HTTP-aware client applications such as web browsers, web service clients, or systems that leverage or extend the HTTP protocol for distributed communication.

HttpClient 组件是为功能扩展而设计的，同时为基本 HTTP 协议提供强有力的支持，任何使用 HTTP 构建的客户端应用程序（如 web 浏览器、web 服务客户端或利用、扩展 HTTP 协议进行分布式通信的系统）的人都可能对它感兴趣。

There are many projects that use HttpClient to provide the core HTTP functionality. Some of these are open source with project pages you can find on the web while others are closed source that you would never see or hear about. The Apache Source License provides maximum flexibility for source and binary reuse. Please see the Applications page for projects using HttpClient.

有许多项目使用 HttpClient 来提供核心的 HTTP 功能。其中一些是开源的，可以在 web 上找到项目页面，而另一些是闭源的，你永远不会看到或听说过。Apache Source License 为源代码和二进制代码的重用提供了最大的灵活性。请参阅使用 HttpClient 的项目的应用程序页面。

## History（历史）

HttpClient was started in 2001 as a subproject of the Jakarta Commons, based on code developed by the Jakarta Slide project. It was promoted out of the Commons in 2004, graduating to a separate Jakarta project. In 2005, the HttpComponents project at Jakarta was created, with the task of developing a successor to HttpClient 3.x and to maintain the existing codebase until the new one is ready to take over. The Commons project, cradle of HttpClient, left Jakarta in 2007 to become an independent Top Level Project. Later in the same year, the HttpComponents project also left Jakarta to become an independent Top Level Project, taking the responsibility for maintaining HttpClient with it.

HttpClient 创建于 2001 年，是 Jakarta Commons 的一个子项目，基于 Jakarta Slide 项目开发。它于 2004 年从 Commons 中提取出来，并逐渐成为 Jakarta 的一个独立项目。2005 年，在 Jakarta 创建了 HttpComponents 项目，其任务是开发 HttpClient 3 的继承者。并维护现有的代码库，直到新的代码库可以接管为止。Commons 项目（HttpClient 的摇篮）在 2007 年脱离了 Jakarta，成为一个独立的顶级项目。同年晚些时候，HttpComponents 项目也脱离了 Jakarta，成为一个独立的顶级项目，负责维护 HttpClient。
