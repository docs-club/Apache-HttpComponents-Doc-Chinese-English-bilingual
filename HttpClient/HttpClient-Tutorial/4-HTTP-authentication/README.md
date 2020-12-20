# Chapter 4 HTTP authentication

**HTTP 身份验证**

HttpClient provides full support for authentication schemes defined by the HTTP standard specification as well as a number of widely used non-standard authentication schemes such as NTLM and SPNEGO.

HttpClient 为 HTTP 标准规范定义的认证模式以及许多广泛使用的非标准认证模式（如 NTLM 和 SPNEGO）提供了全面支持。

## 4.1 User credentials

**用户凭证**

Any process of user authentication requires a set of credentials that can be used to establish user identity. In the simplest form user credentials can be just a user name / password pair. UsernamePasswordCredentials represents a set of credentials consisting of a security principal and a password in clear text. This implementation is sufficient for standard authentication schemes defined by the HTTP standard specification.

任何用户身份验证过程都需要一组凭证，这些凭证可用于建立用户身份。在最简单的情形中，用户凭证可以仅仅是一个用户名/密码对。UsernamePasswordCredentials 表示一组凭证，由安全主体和明文密码组成。这个实现对于 HTTP 标准规范定义的标准身份验证模式来说已经足够了。

```
UsernamePasswordCredentials creds = new UsernamePasswordCredentials("user", "pwd");
System.out.println(creds.getUserPrincipal().getName());
System.out.println(creds.getPassword());
```

stdout >

```
user
pwd
```

NTCredentials is a Microsoft Windows specific implementation that includes in addition to the user name / password pair a set of additional Windows specific attributes such as the name of the user domain. In a Microsoft Windows network the same user can belong to multiple domains each with a different set of authorizations.

NTCredentials 是 Microsoft Windows 特有的实现，除了用户名/密码对外，还包括一组额外的 Windows 特有属性，如用户域的名称。在 Microsoft Windows 网络中，同一用户可以属于多个域，每个域具有一组不同的授权。

```
NTCredentials creds = new NTCredentials("user", "pwd", "workstation", "domain");
System.out.println(creds.getUserPrincipal().getName());
System.out.println(creds.getPassword());
```

stdout >

```
DOMAIN/user
pwd
```

## 4.2 Authentication schemes

**身份验证方案**

The AuthScheme interface represents an abstract challenge-response oriented authentication scheme. An authentication scheme is expected to support the following functions:

AuthScheme 接口代表了一个抽象的面向 challenge-response 的身份验证方案。认证计划应支持下列功能:

- Parse and process the challenge sent by the target server in response to request for a protected resource.

解析和处理目标服务器为响应对受保护资源的请求而发送的质询。

- Provide properties of the processed challenge: the authentication scheme type and its parameters, such the realm this authentication scheme is applicable to, if available

提供已处理的 challenge 的属性：身份验证模式类型及其参数，如果可用的话，提供此身份验证模式适用的领域。

- Generate the authorization string for the given set of credentials and the HTTP request in response to the actual authorization challenge.

为给定的凭据集和 HTTP 请求生成授权字符串，以响应实际的授权 challenge

Please note that authentication schemes may be stateful involving a series of challenge-response exchanges.

请注意，身份验证模式可能是有状态的，涉及一系列 challenge-response 交换。

HttpClient ships with several AuthScheme implementations:

HttpClient 附带了几个 AuthScheme 实现：

- Basic: Basic authentication scheme as defined in RFC 2617. This authentication scheme is insecure, as the credentials are transmitted in clear text. Despite its insecurity Basic authentication scheme is perfectly adequate if used in combination with the TLS/SSL encryption.

Basic：RFC 2617 中定义的基本认证方案。这种身份验证方案是不安全的，因为凭据是以明文传输的。尽管基本身份验证方案不安全，但如果与 TLS/SSL 加密结合使用，它是完全够用的。

- Digest. Digest authentication scheme as defined in RFC 2617. Digest authentication scheme is significantly more secure than Basic and can be a good choice for those applications that do not want the overhead of full transport security through TLS/SSL encryption.

Digest：摘要认证方案定义在 RFC 2617。摘要身份验证方案比基本身份验证方案安全得多，对于那些不希望通过 TLS/SSL 加密实现完全传输安全性的应用程序来说，它是一个很好的选择。

- NTLM: NTLM is a proprietary authentication scheme developed by Microsoft and optimized for Windows platforms. NTLM is believed to be more secure than Digest.

NTLM：NTLM 是微软开发的专有认证方案，为 Windows 平台进行了优化。NTLM 被认为比 Digest 更安全。

- SPNEGO: SPNEGO (Simple and Protected GSSAPI Negotiation Mechanism) is a GSSAPI "pseudo mechanism" that is used to negotiate one of a number of possible real mechanisms. SPNEGO's most visible use is in Microsoft's HTTP Negotiate authentication extension. The negotiable sub-mechanisms include NTLM and Kerberos supported by Active Directory. At present HttpClient only supports the Kerberos sub-mechanism.

SPNEGO：SPNEGO（简单且受保护的 GSSAPI 协商机制）是一个 GSSAPI 「伪机制」，用于协商许多可能的真实机制之一。SPNEGO 最明显的用途是微软的 HTTP 协商身份验证扩展。可协商的子机制包括 Active Directory 支持的 NTLM 和 Kerberos。目前，HttpClient 只支持 Kerberos 子机制。

- Kerberos: Kerberos authentication implementation.

Kerberos：Kerberos 身份验证实现。

## 4.3 Credentials provider 

**凭据提供者**

Credentials providers are intended to maintain a set of user credentials and to be able to produce user credentials for a particular authentication scope. Authentication scope consists of a host name, a port number, a realm name and an authentication scheme name. When registering credentials with the credentials provider one can provide a wild card (any host, any port, any realm, any scheme) instead of a concrete attribute value. The credentials provider is then expected to be able to find the closest match for a particular scope if the direct match cannot be found.

凭据提供者旨在维护一组用户凭据，并能够为特定的身份验证范围生成用户凭据。身份验证范围由主机名、端口号、域名和身份验证方案名组成。在向凭证提供者注册凭证时，可以提供通配符（任何主机、任何端口、任何领域、任何方案），而不是具体的属性值。如果无法找到直接匹配，则期望凭据提供程序能够为特定范围找到最接近的匹配。

HttpClient can work with any physical representation of a credentials provider that implements the CredentialsProvider interface. The default CredentialsProvider implementation called BasicCredentialsProvider is a simple implementation backed by a java.util.HashMap.

HttpClient 可以与实现 CredentialsProvider 接口的凭证提供者的任何物理表示一起工作。名为 BasicCredentialsProvider 的默认 CredentialsProvider 实现是一个简单的实现，它由 java.util.HashMap 支持。

```
CredentialsProvider credsProvider = new BasicCredentialsProvider();
credsProvider.setCredentials(
    new AuthScope("somehost", AuthScope.ANY_PORT),
    new UsernamePasswordCredentials("u1", "p1"));
credsProvider.setCredentials(
    new AuthScope("somehost", 8080),
    new UsernamePasswordCredentials("u2", "p2"));
credsProvider.setCredentials(
    new AuthScope("otherhost", 8080, AuthScope.ANY_REALM, "ntlm"),
    new UsernamePasswordCredentials("u3", "p3"));

System.out.println(credsProvider.getCredentials(
    new AuthScope("somehost", 80, "realm", "basic")));
System.out.println(credsProvider.getCredentials(
    new AuthScope("somehost", 8080, "realm", "basic")));
System.out.println(credsProvider.getCredentials(
    new AuthScope("otherhost", 8080, "realm", "basic")));
System.out.println(credsProvider.getCredentials(
    new AuthScope("otherhost", 8080, null, "ntlm")));
```

stdout >

```
[principal: u1]
[principal: u2]
null
[principal: u3]
```

## 4.4 HTTP authentication and execution context

**HTTP 身份验证和执行上下文**

HttpClient relies on the AuthState class to keep track of detailed information about the state of the authentication process. HttpClient creates two instances of AuthState in the course of HTTP request execution: one for target host authentication and another one for proxy authentication. In case the target server or the proxy require user authentication the respective AuthScope instance will be populated with the AuthScope, AuthScheme and Crednetials used during the authentication process. The AuthState can be examined in order to find out what kind of authentication was requested, whether a matching AuthScheme implementation was found and whether the credentials provider managed to find user credentials for the given authentication scope.

HttpClient 依赖于 AuthState 类来跟踪身份验证过程状态的详细信息。HttpClient 在 HTTP 请求执行过程中创建两个 AuthState 实例：一个用于目标主机身份验证，另一个用于代理身份验证。如果目标服务器或代理需要用户身份验证，则将使用 AuthScope、AuthScheme 和 Crednetials 填充各自的 AuthScope 实例。可以查看 AuthState，以了解请求的身份验证类型，是否找到了匹配的 AuthScheme 实现，以及凭证提供者是否设法为给定的身份验证范围找到了用户凭证。

In the course of HTTP request execution HttpClient adds the following authentication related objects to the execution context:

在 HTTP 请求执行的过程中，HttpClient 将以下与身份验证相关的对象添加到执行上下文：

- Lookup instance representing the actual authentication scheme registry. The value of this attribute set in the local context takes precedence over the default one.

注册中心查找实例代表实际的身份验证方案。在本地上下文中设置的这个属性值优先于默认值。

- CredentialsProvider instance representing the actual credentials provider. The value of this attribute set in the local context takes precedence over the default one.

表示实际凭据提供程序的 CredentialsProvider 实例。在本地上下文中设置的这个属性值优先于默认值。

- AuthState instance representing the actual target authentication state. The value of this attribute set in the local context takes precedence over the default one.

表示实际目标身份验证状态的 AuthState 实例。在本地上下文中设置的这个属性值优先于默认值。

- AuthState instance representing the actual proxy authentication state. The value of this attribute set in the local context takes precedence over the default one.

表示实际的代理身份验证状态的 AuthState 实例。在本地上下文中设置的这个属性值优先于默认值。

- AuthCache instance representing the actual authentication data cache. The value of this attribute set in the local context takes precedence over the default one.

表示实际身份验证数据缓存的 AuthCache 实例。在本地上下文中设置的这个属性值优先于默认值。

The local HttpContext object can be used to customize the HTTP authentication context prior to request execution, or to examine its state after the request has been executed:

本地 HttpContext 对象可以用来在请求执行之前自定义 HTTP 认证上下文，或者在请求执行后检查其状态：

```
CloseableHttpClient httpclient = <...>

CredentialsProvider credsProvider = <...>
Lookup<AuthSchemeProvider> authRegistry = <...>
AuthCache authCache = <...>

HttpClientContext context = HttpClientContext.create();
context.setCredentialsProvider(credsProvider);
context.setAuthSchemeRegistry(authRegistry);
context.setAuthCache(authCache);
HttpGet httpget = new HttpGet("http://somehost/");
CloseableHttpResponse response1 = httpclient.execute(httpget, context);
<...>

AuthState proxyAuthState = context.getProxyAuthState();
System.out.println("Proxy auth state: " + proxyAuthState.getState());
System.out.println("Proxy auth scheme: " + proxyAuthState.getAuthScheme());
System.out.println("Proxy auth credentials: " + proxyAuthState.getCredentials());
AuthState targetAuthState = context.getTargetAuthState();
System.out.println("Target auth state: " + targetAuthState.getState());
System.out.println("Target auth scheme: " + targetAuthState.getAuthScheme());
System.out.println("Target auth credentials: " + targetAuthState.getCredentials());
```

## 4.5 Caching of authentication data

**身份验证数据缓存**

As of version 4.1 HttpClient automatically caches information about hosts it has successfully authenticated with. Please note that one must use the same execution context to execute logically related requests in order for cached authentication data to propagate from one request to another. Authentication data will be lost as soon as the execution context goes out of scope.

从 4.1 版本开始，HttpClient 自动缓存关于它已成功进行身份验证的主机的信息。请注意，必须使用相同的执行上下文来执行逻辑相关的请求，以便缓存的身份验证数据从一个请求传播到另一个请求。一旦执行上下文超出作用域，身份验证数据就会丢失。

## 4.6 Preemptive authentication

**抢先认证**

HttpClient does not support preemptive authentication out of the box, because if misused or used incorrectly the preemptive authentication can lead to significant security issues, such as sending user credentials in clear text to an unauthorized third party. Therefore, users are expected to evaluate potential benefits of preemptive authentication versus security risks in the context of their specific application environment.

HttpClient 不支持开箱即用的抢占式身份验证，因为如果被误用或使用不当，抢先身份验证可能会导致严重的安全问题，比如将用户凭据以明文形式发送给未经授权的第三方。因此，我们希望用户在特定的应用程序环境中评估抢先身份验证的潜在好处与安全风险。

Nonetheless one can configure HttpClient to authenticate preemptively by prepopulating the authentication data cache.

尽管如此，我们可以配置 HttpClient，通过预填充身份验证数据缓存进行抢先身份验证。

```
CloseableHttpClient httpclient = <...>

HttpHost targetHost = new HttpHost("localhost", 80, "http");
CredentialsProvider credsProvider = new BasicCredentialsProvider();
credsProvider.setCredentials(
        new AuthScope(targetHost.getHostName(), targetHost.getPort()),
        new UsernamePasswordCredentials("username", "password"));

// Create AuthCache instance
AuthCache authCache = new BasicAuthCache();
// Generate BASIC scheme object and add it to the local auth cache
BasicScheme basicAuth = new BasicScheme();
authCache.put(targetHost, basicAuth);

// Add AuthCache to the execution context
HttpClientContext context = HttpClientContext.create();
context.setCredentialsProvider(credsProvider);
context.setAuthCache(authCache);

HttpGet httpget = new HttpGet("/");
for (int i = 0; i < 3; i++) {
    CloseableHttpResponse response = httpclient.execute(
            targetHost, httpget, context);
    try {
        HttpEntity entity = response.getEntity();

    } finally {
        response.close();
    }
}
```

## 4.7 NTLM Authentication

As of version 4.1 HttpClient provides full support for NTLMv1, NTLMv2, and NTLM2 Session authentication out of the box. One can still continue using an external NTLM engine such as JCIFS library developed by the Samba project as a part of their Windows interoperability suite of programs.

### 4.7.1 NTLM connection persistence

The NTLM authentication scheme is significantly more expensive in terms of computational overhead and performance impact than the standard Basic and Digest schemes. This is likely to be one of the main reasons why Microsoft chose to make NTLM authentication scheme stateful. That is, once authenticated, the user identity is associated with that connection for its entire life span. The stateful nature of NTLM connections makes connection persistence more complex, as for the obvious reason persistent NTLM connections may not be re-used by users with a different user identity. The standard connection managers shipped with HttpClient are fully capable of managing stateful connections. However, it is critically important that logically related requests within the same session use the same execution context in order to make them aware of the current user identity. Otherwise, HttpClient will end up creating a new HTTP connection for each HTTP request against NTLM protected resources. For detailed discussion on stateful HTTP connections please refer to this section.

As NTLM connections are stateful it is generally recommended to trigger NTLM authentication using a relatively cheap method, such as GET or HEAD, and re-use the same connection to execute more expensive methods, especially those enclose a request entity, such as POST or PUT.

```
CloseableHttpClient httpclient = <...>

CredentialsProvider credsProvider = new BasicCredentialsProvider();
credsProvider.setCredentials(AuthScope.ANY,
        new NTCredentials("user", "pwd", "myworkstation", "microsoft.com"));

HttpHost target = new HttpHost("www.microsoft.com", 80, "http");

// Make sure the same context is used to execute logically related requests
HttpClientContext context = HttpClientContext.create();
context.setCredentialsProvider(credsProvider);

// Execute a cheap method first. This will trigger NTLM authentication
HttpGet httpget = new HttpGet("/ntlm-protected/info");
CloseableHttpResponse response1 = httpclient.execute(target, httpget, context);
try {
    HttpEntity entity1 = response1.getEntity();
} finally {
    response1.close();
}

// Execute an expensive method next reusing the same context (and connection)
HttpPost httppost = new HttpPost("/ntlm-protected/form");
httppost.setEntity(new StringEntity("lots and lots of data"));
CloseableHttpResponse response2 = httpclient.execute(target, httppost, context);
try {
    HttpEntity entity2 = response2.getEntity();
} finally {
    response2.close();
}
```

## 4.8 SPNEGO/Kerberos Authentication

The SPNEGO (Simple and Protected GSSAPI Negotiation Mechanism) is designed to allow for authentication to services when neither end knows what the other can use/provide. It is most commonly used to do Kerberos authentication. It can wrap other mechanisms, however the current version in HttpClient is designed solely with Kerberos in mind.

- Client Web Browser does HTTP GET for resource.

- Web server returns HTTP 401 status and a header: WWW-Authenticate: Negotiate

- Client generates a NegTokenInit, base64 encodes it, and resubmits the GET with an Authorization header: Authorization: `Negotiate <base64 encoding>`.

- Server decodes the NegTokenInit, extracts the supported MechTypes (only Kerberos V5 in our case), ensures it is one of the expected ones, and then extracts the MechToken (Kerberos Token) and authenticates it.

If more processing is required another HTTP 401 is returned to the client with more data in the the WWW-Authenticate header. Client takes the info and generates another token passing this back in the Authorization header until complete.

- When the client has been authenticated the Web server should return the HTTP 200 status, a final WWW-Authenticate header and the page content.

当客户端已通过身份验证时，Web 服务器应该返回 HTTP 200 状态，一个最终的 WWW-Authenticate 头和页面内容。

### 4.8.1 SPNEGO support in HttpClient

The SPNEGO authentication scheme is compatible with Sun Java versions 1.5 and up. However the use of Java >= 1.6 is strongly recommended as it supports SPNEGO authentication more completely.

The Sun JRE provides the supporting classes to do nearly all the Kerberos and SPNEGO token handling. This means that a lot of the setup is for the GSS classes. The SPNegoScheme is a simple class to handle marshalling the tokens and reading and writing the correct headers.

The best way to start is to grab the KerberosHttpClient.java file in examples and try and get it to work. There are a lot of issues that can happen but if lucky it'll work without too much of a problem. It should also provide some output to debug with.

In Windows it should default to using the logged in credentials; this can be overridden by using 'kinit' e.g. \$JAVA_HOME\bin\kinit testuser@AD.EXAMPLE.NET, which is very helpful for testing and debugging issues. Remove the cache file created by kinit to revert back to the windows Kerberos cache.

Make sure to list domain_realms in the krb5.conf file. This is a major source of problems.

### 4.8.2 GSS/Java Kerberos Setup

This documentation assumes you are using Windows but much of the information applies to Unix as well.

The org.ietf.jgss classes have lots of possible configuration parameters, mainly in the krb5.conf/krb5.ini file. Some more info on the format at http://web.mit.edu/kerberos/krb5-1.4/krb5-1.4.1/doc/krb5-admin/krb5.conf.html.

### 4.8.3 login.conf file

The following configuration is a basic setup that works in Windows XP against both IIS and JBoss Negotiation modules.

The system property java.security.auth.login.config can be used to point at the login.conf file.

login.conf content may look like the following:

```
com.sun.security.jgss.login {
  com.sun.security.auth.module.Krb5LoginModule required client=TRUE useTicketCache=true;
};

com.sun.security.jgss.initiate {
  com.sun.security.auth.module.Krb5LoginModule required client=TRUE useTicketCache=true;
};

com.sun.security.jgss.accept {
  com.sun.security.auth.module.Krb5LoginModule required client=TRUE useTicketCache=true;
};
```

### 4.8.4 krb5.conf / krb5.ini file

If unspecified, the system default will be used. Override if needed by setting the system property java.security.krb5.conf to point to a custom krb5.conf file.

krb5.conf content may look like the following:

```
[libdefaults]
    default_realm = AD.EXAMPLE.NET
    udp_preference_limit = 1
[realms]
    AD.EXAMPLE.NET = {
        kdc = KDC.AD.EXAMPLE.NET
    }
[domain_realms]
.ad.example.net=AD.EXAMPLE.NET
ad.example.net=AD.EXAMPLE.NET
```

### 4.8.5 Windows Specific configuration

To allow Windows to use the current user's tickets, the system property javax.security.auth.useSubjectCredsOnly must be set to false and the Windows registry key allowtgtsessionkey should be added and set correctly to allow session keys to be sent in the Kerberos Ticket-Granting Ticket.

On the Windows Server 2003 and Windows 2000 SP4, here is the required registry setting:

```
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Lsa\Kerberos\Parameters
Value Name: allowtgtsessionkey
Value Type: REG_DWORD
Value: 0x01
```

Here is the location of the registry setting on Windows XP SP2:

```
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Lsa\Kerberos\
Value Name: allowtgtsessionkey
Value Type: REG_DWORD
Value: 0x01
```

---

**[Back to contents of HttpClient Tutorial（返回 HttpClient 教程目录）](https://github.com/clxering/Apache-HttpComponents-Doc-Chinese-English-bilingual/tree/master/HttpClient/HttpClient-Tutorial#contents)**

- **Previous Chapter：[Chapter 3 HTTP state management](https://github.com/clxering/Apache-HttpComponents-Doc-Chinese-English-bilingual/tree/master/HttpClient/HttpClient-Tutorial/3-HTTP-state-management#chapter-3-http-state-management)**
- **Next Chapter：[Chapter 5 Fluent API](https://github.com/clxering/Apache-HttpComponents-Doc-Chinese-English-bilingual/tree/master/HttpClient/HttpClient-Tutorial/5-Fluent-API#chapter-5-fluent-api)**
