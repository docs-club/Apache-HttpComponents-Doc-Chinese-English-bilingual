## Preface
The Hyper-Text Transfer Protocol (HTTP) is perhaps the most significant protocol used on the Internet today. Web services, network-enabled appliances and the growth of network computing continue to expand the role of the HTTP protocol beyond user-driven web browsers, while increasing the number of applications that require HTTP support.

Although the java.net package provides basic functionality for accessing resources via HTTP, it doesn't provide the full flexibility or functionality needed by many applications. HttpClient seeks to fill this void by providing an efficient, up-to-date, and feature-rich package implementing the client side of the most recent HTTP standards and recommendations.

Designed for extension while providing robust support for the base HTTP protocol, HttpClient may be of interest to anyone building HTTP-aware client applications such as web browsers, web service clients, or systems that leverage or extend the HTTP protocol for distributed communication.

### 1. HttpClient scope
Client-side HTTP transport library based on HttpCore

Based on classic (blocking) I/O

Content agnostic

### 2. What HttpClient is NOT
HttpClient is NOT a browser. It is a client side HTTP transport library. HttpClient's purpose is to transmit and receive HTTP messages. HttpClient will not attempt to process content, execute javascript embedded in HTML pages, try to guess content type, if not explicitly set, or reformat request / rewrite location URIs, or other functionality unrelated to the HTTP transport.
