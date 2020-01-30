# Chapter 3 HTTP state management

Originally HTTP was designed as a stateless, request / response oriented protocol that made no special provisions for stateful sessions spanning across several logically related request / response exchanges. As HTTP protocol grew in popularity and adoption more and more systems began to use it for applications it was never intended for, for instance as a transport for e-commerce applications. Thus, the support for state management became a necessity.

最初，HTTP 被设计为一种无状态的、面向请求/响应的协议，它没有为跨越几个逻辑相关的请求/响应交换的有状态会话做出特殊规定。随着 HTTP 协议的流行和广泛使用，越来越多的系统开始将其用于前所未有的应用程序，例如作为电子商务应用程序的传输。因此，支持状态管理成为必要。

Netscape Communications, at that time a leading developer of web client and server software, implemented support for HTTP state management in their products based on a proprietary specification. Later, Netscape tried to standardise the mechanism by publishing a specification draft. Those efforts contributed to the formal specification defined through the RFC standard track. However, state management in a significant number of applications is still largely based on the Netscape draft and is incompatible with the official specification. All major developers of web browsers felt compelled to retain compatibility with those applications greatly contributing to the fragmentation of standards compliance.

Netscape Communications 当时是 web 客户端和服务器软件的引领者，在其产品中基于专有规范实现了对 HTTP 状态管理的支持。后来，Netscape 试图通过发布规范草案来标准化该机制。这些工作促成了通过 RFC 标准跟踪定义的正式规范。然而，大量应用程序中的状态管理仍然主要基于 Netscape 草案，并且与官方规范不兼容。web 浏览器的所有主要开发人员都感到必须保持与这些应用程序的兼容性，这大大加剧了标准遵从性的碎片化。

## 3.1 HTTP cookies

An HTTP cookie is a token or short packet of state information that the HTTP agent and the target server can exchange to maintain a session. Netscape engineers used to refer to it as a "magic cookie" and the name stuck.

HTTP cookie 是 HTTP 代理和目标服务器可以交换的状态信息的令牌或短包，用于维护会话。网景（Netscape）的工程师曾把它称为「魔法饼干」（magic cookie），并沿用了这个名字。

HttpClient uses the Cookie interface to represent an abstract cookie token. In its simplest form an HTTP cookie is merely a name / value pair. Usually an HTTP cookie also contains a number of attributes such a domain for which is valid, a path that specifies the subset of URLs on the origin server to which this cookie applies, and the maximum period of time for which the cookie is valid.

HttpClient 使用 Cookie 接口来表示抽象的 Cookie 令牌。HTTP cookie 最简单的形式就是一个名称/值对。通常，HTTP cookie 还包含许多属性，比如一个有效的域、一个指定该 cookie 应用于的源服务器上 URL 子集的路径，以及该 cookie 有效的最大时间段。

The SetCookie interface represents a Set-Cookie response header sent by the origin server to the HTTP agent in order to maintain a conversational state.

SetCookie 接口表示源服务器发送给 HTTP 代理的一个 Set-Cookie 响应头，用于维护会话状态。

The ClientCookie interface extends Cookie interface with additional client specific functionality such as the ability to retrieve original cookie attributes exactly as they were specified by the origin server. This is important for generating the Cookie header because some cookie specifications require that the Cookie header should include certain attributes only if they were specified in the Set-Cookie header.

ClientCookie 接口使用附加的客户端特定功能扩展了 Cookie 接口，比如能够按照原始服务器指定的方式检索原始 Cookie 属性。这对于生成 Cookie 头非常重要，因为一些 Cookie 规范要求只有在 Set-Cookie 头中指定某些属性时，Cookie 头才应该包含这些属性。

Here is an example of creating a client-side cookie object:

下面是一个创建客户端 cookie 对象的例子：

```
BasicClientCookie cookie = new BasicClientCookie("name", "value");
// Set effective domain and path attributes
cookie.setDomain(".mycompany.com");
cookie.setPath("/");
// Set attributes exactly as sent by the server
cookie.setAttribute(ClientCookie.PATH_ATTR, "/");
cookie.setAttribute(ClientCookie.DOMAIN_ATTR, ".mycompany.com");
```

## 3.2 Cookie specifications

The CookieSpec interface represents a cookie management specification. The cookie management specification is expected to enforce:

CookieSpec 接口表示一个 cookie 管理规范。要求 cookie 管理规范强制执行：

- rules of parsing Set-Cookie headers.

解析 Set-Cookie 头的规则。

- rules of validation of parsed cookies.

已解析 cookie 的验证规则。

- formatting of Cookie header for a given host, port and path of origin.

为给定的主机、端口和原始路径格式化 Cookie 头。

HttpClient ships with several CookieSpec implementations:

HttpClient 附带了几个 CookieSpec 实现：

- **Standard strict:** State management policy compliant with the syntax and semantics of the well-behaved profile defined by RFC 6265, section 4.

Standard strict：符合 RFC 6265 第 4 节定义，并且行为良好的概要文件的语法和语义的状态管理策略。

- **Standard:** State management policy compliant with a more relaxed profile defined by RFC 6265, section 4 intended for interoperability with existing servers that do not conform to the well behaved profile.

Standard：符合 RFC 6265 定义的更宽松概要文件的状态管理策略，第 4 节旨在与不符合行为良好概要文件的现有服务器进行互操作性。

- **Netscape draft (obsolete):** This policy conforms to the original draft specification published by Netscape Communications. It should be avoided unless absolutely necessary for compatibility with legacy code.

Netscape draft（已过时）：此策略符合 Netscape Communications 发布的原始规范草案。除非绝对需要与遗留代码兼容，否则应该避免使用它。

- **RFC 2965 (obsolete):** State management policy compliant with the obsolete state management specification defined by RFC 2965. Please do not use in new applications.

RFC 2965（过时）：符合 RFC 2965 定义的过时状态管理规范的状态管理策略。请不要在新的应用程序中使用。

- **RFC 2109 (obsolete):** State management policy compliant with the obsolete state management specification defined by RFC 2109. Please do not use in new applications.

RFC 2109（过时）：符合 RFC 2109 定义的过时状态管理规范的状态管理策略。请不要在新的应用程序中使用。

- **Browser compatibility (obsolete):** This policy strives to closely mimic the (mis)behavior of older versions of browser applications such as Microsoft Internet Explorer and Mozilla FireFox. Please do not use in new applications.

浏览器兼容性（已过时）：此策略力求紧密模仿旧版本浏览器应用程序（mis）的行为，如 Microsoft Internet Explorer 和 Mozilla FireFox。请不要在新的应用程序中使用。

- **Default:** Default cookie policy is a synthetic policy that picks up either RFC 2965, RFC 2109 or Netscape draft compliant implementation based on properties of cookies sent with the HTTP response (such as version attribute, now obsolete). This policy will be deprecated in favor of the standard (RFC 6265 compliant) implementation in the next minor release of HttpClient.

默认：默认 cookie 策略是一种综合策略，它根据 HTTP 响应发送的 cookie 的属性（例如 version 属性，现在已经过时）选择 RFC 2965、RFC 2109 或符合 Netscape draft 的实现。此策略将在 HttpClient 的下一个小版本中被弃用，取而代之的是标准（RFC 6265 兼容）实现。

- **Ignore cookies:** All cookies are ignored.

忽略 cookie：忽略所有 cookie。

It is strongly recommended to use either Standard or Standard strict policy in new applications. Obsolete specifications should be used for compatibility with legacy systems only. Support for obsolete specifications will be removed in the next major release of HttpClient.

强烈建议在新应用程序中使用 Standard 或 Standard strict 策略。过时的规范应该只用于与遗留系统兼容。对过时规范的支持将在 HttpClient 的下一个主要版本中删除。

## 3.3 Choosing cookie policy

Cookie policy can be set at the HTTP client and overridden on the HTTP request level if required.

Cookie 策略可以在 HTTP 客户端上设置，如果需要，还可以在 HTTP 请求级别上重写。

```
RequestConfig globalConfig = RequestConfig.custom()
        .setCookieSpec(CookieSpecs.DEFAULT)
        .build();
CloseableHttpClient httpclient = HttpClients.custom()
        .setDefaultRequestConfig(globalConfig)
        .build();
RequestConfig localConfig = RequestConfig.copy(globalConfig)
        .setCookieSpec(CookieSpecs.STANDARD_STRICT)
        .build();
HttpGet httpGet = new HttpGet("/");
httpGet.setConfig(localConfig);
```

## 3.4 Custom cookie policy

In order to implement a custom cookie policy one should create a custom implementation of the CookieSpec interface, create a CookieSpecProvider implementation to create and initialize instances of the custom specification and register the factory with HttpClient. Once the custom specification has been registered, it can be activated the same way as a standard cookie specification.

为了实现自定义 cookie 策略，应该创建一个 CookieSpec 接口的自定义实现，创建一个 CookieSpecProvider 实现来创建和初始化自定义规范的实例，并将工厂注册到 HttpClient。一旦注册了自定义规范，就可以像标准 cookie 规范一样激活它。

```
PublicSuffixMatcher publicSuffixMatcher = PublicSuffixMatcherLoader.getDefault();

Registry<CookieSpecProvider> r = RegistryBuilder.<CookieSpecProvider>create()
        .register(CookieSpecs.DEFAULT,
                new DefaultCookieSpecProvider(publicSuffixMatcher))
        .register(CookieSpecs.STANDARD,
                new RFC6265CookieSpecProvider(publicSuffixMatcher))
        .register("easy", new EasySpecProvider())
        .build();

RequestConfig requestConfig = RequestConfig.custom()
        .setCookieSpec("easy")
        .build();

CloseableHttpClient httpclient = HttpClients.custom()
        .setDefaultCookieSpecRegistry(r)
        .setDefaultRequestConfig(requestConfig)
        .build();
```

## 3.5 Cookie persistence

HttpClient can work with any physical representation of a persistent cookie store that implements the CookieStore interface. The default CookieStore implementation called BasicCookieStore is a simple implementation backed by a java.util.ArrayList. Cookies stored in an BasicClientCookie object are lost when the container object get garbage collected. Users can provide more complex implementations if necessary.

HttpClient 可以处理实现 CookieStore 接口的持久 cookie 存储的任何物理表示。默认的 CookieStore 实现名为 BasicCookieStore，是一个由 java.util.ArrayList 支持的简单实现。当容器对象被垃圾收集时，存储在 BasicClientCookie 对象中的 cookie 将丢失。如果需要，用户可以提供更复杂的实现。

```
// Create a local instance of cookie store
CookieStore cookieStore = new BasicCookieStore();
// Populate cookies if needed
BasicClientCookie cookie = new BasicClientCookie("name", "value");
cookie.setDomain(".mycompany.com");
cookie.setPath("/");
cookieStore.addCookie(cookie);
// Set the store
CloseableHttpClient httpclient = HttpClients.custom()
        .setDefaultCookieStore(cookieStore)
        .build();
```

## 3.6 HTTP state management and execution context

In the course of HTTP request execution HttpClient adds the following state management related objects to the execution context:

在 HTTP 请求执行过程中，HttpClient 将以下状态管理相关对象添加到执行 context 中：

- Lookup instance representing the actual cookie specification registry. The value of this attribute set in the local context takes precedence over the default one.

Lookup 实例，表示实际的 cookie 规范注册表。此属性集在本地上下文中的值优先于默认值。

- CookieSpec instance representing the actual cookie specification.

CookieSpec 实例，表示实际的 cookie 规范。

- CookieOrigin instance representing the actual details of the origin server.

CookieOrigin 实例，表示源服务器的实际细节。

- CookieStore instance representing the actual cookie store. The value of this attribute set in the local context takes precedence over the default one.

CookieStore 实例，表示实际的 cookie 存储。此属性集在本地 context 中的值优先于默认值。

The local HttpContext object can be used to customize the HTTP state management context prior to request execution, or to examine its state after the request has been executed. One can also use separate execution contexts in order to implement per user (or per thread) state management. A cookie specification registry and cookie store defined in the local context will take precedence over the default ones set at the HTTP client level

本地 HttpContext 对象可用于在请求执行前定制 HTTP 状态管理 context，或在请求执行后检查其状态。还可以使用单独的执行 context 来实现每个用户（或每个线程）的状态管理。在本地 context 中定义的 cookie 规范注册表和 cookie 存储将优先于在 HTTP 客户端级别设置的默认注册表和 cookie 存储。

```
CloseableHttpClient httpclient = <...>

Lookup<CookieSpecProvider> cookieSpecReg = <...>
CookieStore cookieStore = <...>

HttpClientContext context = HttpClientContext.create();
context.setCookieSpecRegistry(cookieSpecReg);
context.setCookieStore(cookieStore);
HttpGet httpget = new HttpGet("http://somehost/");
CloseableHttpResponse response1 = httpclient.execute(httpget, context);
<...>
// Cookie origin details
CookieOrigin cookieOrigin = context.getCookieOrigin();
// Cookie spec used
CookieSpec cookieSpec = context.getCookieSpec();
```

---

**[Back to contents of HttpClient Tutorial（返回 HttpClient 教程目录）](https://github.com/clxering/Apache-HttpComponents-Doc-Chinese-English-bilingual/tree/master/HttpClient/HttpClient-Tutorial#contents)**

- **Previous Chapter：[Chapter 2 Connection management](https://github.com/clxering/Apache-HttpComponents-Doc-Chinese-English-bilingual/tree/master/HttpClient/HttpClient-Tutorial/2-Connection-management#chapter-2-connection-management)**
- **Next Chapter：[Chapter 4 HTTP authentication](https://github.com/clxering/Apache-HttpComponents-Doc-Chinese-English-bilingual/tree/master/HttpClient/HttpClient-Tutorial/4-HTTP-authentication#chapter-4-http-authentication)**
