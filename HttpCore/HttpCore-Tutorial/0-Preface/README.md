# Preface（前言）

HttpCore is a set of components implementing the most fundamental aspects of the HTTP protocol that are nonetheless sufficient to develop full-featured client-side and server-side HTTP services with a minimal footprint.

HttpCore 是一组实现 HTTP 协议最基本方面的组件，尽管如此，它仍然足以以最小的代价开发功能齐全的客户端和服务器端 HTTP 服务。

HttpCore has the following scope and goals:

HttpCore 有以下范围和目标：

1. HttpCore Scope（HttpCore 范围）

- A consistent API for building client / proxy / server side HTTP services

用于构建客户端/代理端/服务器端 HTTP 服务的一致 API

- A consistent API for building both synchronous and asynchronous HTTP services

用于构建同步和异步 HTTP 服务的一致 API

- A set of low level components based on blocking (classic) and non-blocking (NIO) I/O models

一组基于阻塞（classic）和非阻塞（NIO）I/O 模型的低层组件

2. HttpCore Goals（HttpCore 目标）

- Implementation of the most fundamental HTTP transport aspects

实现最基本的 HTTP 传输方面

- Balance between good performance and the clarity & expressiveness of API

平衡良好的性能和 API 的清晰度和表现力

- Small (predictable) memory footprint

很小（可预测）的内存占用

- Self-contained library (no external dependencies beyond JRE)

自包含的库（除了 JRE 之外没有外部依赖项）

3. What HttpCore is NOT（HttpCore 不是什么）

- A replacement for HttpClient

HttpClient 的替代品

- A replacement for Servlet APIs

Servlet API 的替代品
