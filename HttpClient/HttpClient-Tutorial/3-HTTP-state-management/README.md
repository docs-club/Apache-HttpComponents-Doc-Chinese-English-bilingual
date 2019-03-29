## Chapter 3. HTTP state management
Originally HTTP was designed as a stateless, request / response oriented protocol that made no special provisions for stateful sessions spanning across several logically related request / response exchanges. As HTTP protocol grew in popularity and adoption more and more systems began to use it for applications it was never intended for, for instance as a transport for e-commerce applications. Thus, the support for state management became a necessity.

Netscape Communications, at that time a leading developer of web client and server software, implemented support for HTTP state management in their products based on a proprietary specification. Later, Netscape tried to standardise the mechanism by publishing a specification draft. Those efforts contributed to the formal specification defined through the RFC standard track. However, state management in a significant number of applications is still largely based on the Netscape draft and is incompatible with the official specification. All major developers of web browsers felt compelled to retain compatibility with those applications greatly contributing to the fragmentation of standards compliance.

### 3.1 HTTP cookies
An HTTP cookie is a token or short packet of state information that the HTTP agent and the target server can exchange to maintain a session. Netscape engineers used to refer to it as a "magic cookie" and the name stuck.

HttpClient uses the Cookie interface to represent an abstract cookie token. In its simplest form an HTTP cookie is merely a name / value pair. Usually an HTTP cookie also contains a number of attributes such a domain for which is valid, a path that specifies the subset of URLs on the origin server to which this cookie applies, and the maximum period of time for which the cookie is valid.

The SetCookie interface represents a `Set-Cookie` response header sent by the origin server to the HTTP agent in order to maintain a conversational state.

The ClientCookie interface extends Cookie interface with additional client specific functionality such as the ability to retrieve original cookie attributes exactly as they were specified by the origin server. This is important for generating the Cookie header because some cookie specifications require that the Cookie header should include certain attributes only if they were specified in the `Set-Cookie` header.

Here is an example of creating a client-side cookie object:

```
BasicClientCookie cookie = new BasicClientCookie("name", "value");
// Set effective domain and path attributes
cookie.setDomain(".mycompany.com");
cookie.setPath("/");
// Set attributes exactly as sent by the server
cookie.setAttribute(ClientCookie.PATH_ATTR, "/");
cookie.setAttribute(ClientCookie.DOMAIN_ATTR, ".mycompany.com");
```

### 3.2 Cookie specifications
The `CookieSpec` interface represents a cookie management specification. The cookie management specification is expected to enforce:

- rules of parsing `Set-Cookie` headers.

- rules of validation of parsed cookies.

- formatting of `Cookie` header for a given host, port and path of origin.

HttpClient ships with several `CookieSpec` implementations:

- **Standard strict:**  State management policy compliant with the syntax and semantics of the well-behaved profile defined by RFC 6265, section 4.

- **Standard:**  State management policy compliant with a more relaxed profile defined by RFC 6265, section 4 intended for interoperability with existing servers that do not conform to the well behaved profile.

- **Netscape draft (obsolete):**  This policy conforms to the original draft specification published by Netscape Communications. It should be avoided unless absolutely necessary for compatibility with legacy code.

- **RFC 2965 (obsolete):**  State management policy compliant with the obsolete state management specification defined by RFC 2965. Please do not use in new applications.

- **RFC 2109 (obsolete):**  State management policy compliant with the obsolete state management specification defined by RFC 2109. Please do not use in new applications.

- **Browser compatibility (obsolete):**  This policy strives to closely mimic the (mis)behavior of older versions of browser applications such as Microsoft Internet Explorer and Mozilla FireFox. Please do not use in new applications.

- **Default:**  Default cookie policy is a synthetic policy that picks up either RFC 2965, RFC 2109 or Netscape draft compliant implementation based on properties of cookies sent with the HTTP response (such as version attribute, now obsolete). This policy will be deprecated in favor of the standard (RFC 6265 compliant) implementation in the next minor release of HttpClient.

- **Ignore cookies:**  All cookies are ignored.

It is strongly recommended to use either `Standard` or `Standard strict` policy in new applications. Obsolete specifications should be used for compatibility with legacy systems only. Support for obsolete specifications will be removed in the next major release of HttpClient.

### 3.3 Choosing cookie policy
Cookie policy can be set at the HTTP client and overridden on the HTTP request level if required.

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

### 3.4 Custom cookie policy
In order to implement a custom cookie policy one should create a custom implementation of the `CookieSpec` interface, create a `CookieSpecProvider` implementation to create and initialize instances of the custom specification and register the factory with HttpClient. Once the custom specification has been registered, it can be activated the same way as a standard cookie specification.

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

### 3.5 Cookie persistence
HttpClient can work with any physical representation of a persistent cookie store that implements the `CookieStore` interface. The default `CookieStore` implementation called `BasicCookieStore` is a simple implementation backed by a `java.util.ArrayList`. Cookies stored in an `BasicClientCookie` object are lost when the container object get garbage collected. Users can provide more complex implementations if necessary.

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

### 3.6 HTTP state management and execution context
In the course of HTTP request execution HttpClient adds the following state management related objects to the execution context:

- `Lookup` instance representing the actual cookie specification registry. The value of this attribute set in the local context takes precedence over the default one.

- `CookieSpec` instance representing the actual cookie specification.

- `CookieOrigin` instance representing the actual details of the origin server.

- `CookieStore` instance representing the actual cookie store. The value of this attribute set in the local context takes precedence over the default one.

The local `HttpContext` object can be used to customize the HTTP state management context prior to request execution, or to examine its state after the request has been executed. One can also use separate execution contexts in order to implement per user (or per thread) state management. A cookie specification registry and cookie store defined in the local context will take precedence over the default ones set at the HTTP client level

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
