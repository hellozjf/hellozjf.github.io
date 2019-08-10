# 第四章 HTTP身份认证

HttpClient完全支持HTTP标准规范定义的身份验证方案以及许多广泛使用的非标准身份验证方案，如NTLM和SPNEGO。

## 4.1. 用户凭据

任何用户身份验证过程都需要一组可用于建立用户身份的凭据。 在最简单的形式中，用户凭证可以只是用户名/密码对。 UsernamePasswordCredentials表示由明文组成的安全主体和密码组成的一组凭据。 此实现足以用于HTTP标准规范定义的标准身份验证方案。

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

NTCredentials是Microsoft Windows特定的实现，除了用户名/密码对之外还包括一组其他Windows特定属性，例如用户域的名称。 在Microsoft Windows网络中，同一用户可以属于多个域，每个域具有不同的授权集。

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

## 4.2. 认证方案

AuthScheme接口表示面向抽象挑战 - 响应的身份验证方案。 认证方案有望支持以下功能：

* 解析并处理目标服务器发送的质询，以响应对受保护资源的请求。
* 提供已处理质询的属性：身份验证方案类型及其参数，例如此身份验证方案适用的域（如果可用）
* 为给定的凭证集和HTTP请求生成授权字符串以响应实际的授权质询。

请注意，身份验证方案可能是有状态的，涉及一系列质询 - 响应交换。

HttpClient附带了几个AuthScheme实现：

* Basic：RFC 2617中定义的基本身份验证方案。此身份验证方案不安全，因为凭据以明文形式传输。 尽管不安全，但如果与TLS / SSL加密结合使用，基本身份验证方案就足够了。
* Digest： RFC 2617中定义的摘要式身份验证方案。摘要式身份验证方案比Basic更安全，对于那些不希望通过TLS / SSL加密实现完全传输安全性开销的应用程序而言，它们是一个不错的选择。
* NTLM：NTLM是Microsoft开发的专有身份验证方案，针对Windows平台进行了优化。 NTLM被认为比Digest更安全。
* SPNEGO：SPNEGO（简单和受保护的GSSAPI协商机制）是GSSAPI“伪机制”，用于协商许多可能的实际机制之一。 SPNEGO最明显的用途是在Microsoft的HTTP Negotiate身份验证扩展中。 可协商的子机制包括Active Directory支持的NTLM和Kerberos。 目前HttpClient仅支持Kerberos子机制。
* Kerberos：Kerberos身份验证实现。

## 4.3. 凭证提供程序

凭据提供程序旨在维护一组用户凭据，并能够为特定的身份验证范围生成用户凭据。 身份验证范围包括主机名，端口号，域名和身份验证方案名称。 在凭证提供程序中注册凭据时，可以提供通配符（任何主机，任何端口，任何领域，任何方案）而不是具体的属性值。 如果无法找到直接匹配，则凭证提供者可以找到特定范围的最接近匹配。

HttpClient可以与实现CredentialsProvider接口的凭证提供程序的任何物理表示一起使用。 名为BasicCredentialsProvider的默认CredentialsProvider实现是一个由java.util.HashMap支持的简单实现。

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

## 4.4. HTTP身份验证和执行上下文

HttpClient依赖于AuthState类来跟踪有关身份验证过程状态的详细信息。 HttpClient在HTTP请求执行过程中创建两个AuthState实例：一个用于目标主机身份验证，另一个用于代理身份验证。 如果目标服务器或代理需要用户身份验证，则相应的AuthScope实例将使用身份验证过程中使用的AuthScope，AuthScheme和Crednetials进行填充。 可以检查AuthState以查明请求的身份验证类型，是否找到匹配的AuthScheme实现以及凭据提供程序是否设法查找给定身份验证范围的用户凭据。

在HTTP请求执行过程中，HttpClient将以下与身份验证相关的对象添加到执行上下文中：

* `Lookup`实例表示实际的身份验证方案注册表。 在本地上下文中设置的此属性的值优先于默认值。
* `CredentialsProvider`实例表示实际凭据提供程序。 在本地上下文中设置的此属性的值优先于默认值。
* `AuthState`实例表示实际目标身份验证状态。 在本地上下文中设置的此属性的值优先于默认值。
* `AuthCache`实例表示实际的身份验证数据缓存。 在本地上下文中设置的此属性的值优先于默认值。

本地`HttpContext`对象可用于在请求执行之前自定义HTTP身份验证上下文，或在请求执行后检查其状态：

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

## 4.5. 缓存身份验证数据

从版本4.1开始，HttpClient会自动缓存有关已成功通过身份验证的主机的信息。 请注意，必须使用相同的执行上下文来执行逻辑上相关的请求，以便缓存的身份验证数据从一个请求传播到另一个请求。 一旦执行上下文超出范围，身份验证数据就会丢失。

## 4.6. 抢先认证

HttpClient不支持开箱即用的抢先身份验证，因为如果误用或使用不当，抢先身份验证可能会导致严重的安全问题，例如以明文形式向未经授权的第三方发送用户凭据。 因此，期望用户在其特定应用环境的背景下评估抢先认证与安全风险的潜在好处。

尽管如此，可以通过预先填充身份验证数据缓存来配置HttpClient以预先进行身份验证。

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

## 4.7. NTLM身份验证

从版本4.1开始，HttpClient提供对NTLMv1，NTLMv2和NTLM2会话身份验证的完全支持。 人们仍然可以继续使用外部NTLM引擎，例如Samba项目开发的JCIFS库，作为其Windows互操作性程序套件的一部分。

### 4.7.1. NTLM连接持久性

与标准的Basic和Digest方案相比，NTLM认证方案在计算开销和性能影响方面要贵得多。这可能是微软选择使NTLM身份验证方案成为有状态的主要原因之一。也就是说，一旦经过身份验证，用户身份就会在整个生命周期内与该连接相关联。 NTLM连接的有状态特性使连接持久性更加复杂，因为显而易见的原因是具有不同用户身份的用户可能无法重用持久性NTLM连接。 HttpClient附带的标准连接管理器完全能够管理有状态连接。但是，同一会话中的逻辑相关请求使用相同的执行上下文以使其了解当前用户身份至关重要。否则，HttpClient将最终为针对NTLM保护资源的每个HTTP请求创建新的HTTP连接。有关有状态HTTP连接的详细讨论，请参阅本节。

由于NTLM连接是有状态的，因此通常建议使用相对便宜的方法（如GET或HEAD）触发NTLM身份验证，并重新使用相同的连接来执行更昂贵的方法，尤其是那些包含请求实体的方法，例如POST或PUT。

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

## 4.8. SPNEGO / Kerberos身份验证

SPNEGO（简单和受保护的GSSAPI协商机制）旨在允许在双方都不知道对方可以使用/提供的内容时对服务进行身份验证。 它最常用于执行Kerberos身份验证。 它可以包装其他机制，但是HttpClient中的当前版本仅考虑Kerberos而设计。

### 4.8.1. HttpClient中的SPNEGO支持

SPNEGO身份验证方案与Sun Java 1.5及更高版本兼容。 但是强烈建议使用Java> = 1.6，因为它更完整地支持SPNEGO身份验证。

Sun JRE提供了支持类来执行几乎所有Kerberos和SPNEGO令牌处理。 这意味着很多设置都适用于GSS类。 SPNegoScheme是一个简单的类，用于处理编组令牌以及读取和写入正确的标题。

最好的方法是在示例中获取KerberosHttpClient.java文件并尝试使其工作。 可能会发生很多问题，但如果幸运的话，它可以在没有太多问题的情况下运行。 它还应该提供一些输出来调试。

在Windows中，它应默认使用登录的凭据; 这可以通过使用'kinit'来覆盖，例如 `$JAVA_HOME\bin\kinit testuser@AD.EXAMPLE.NET`，这对测试和调试问题非常有帮助。 删除kinit创建的缓存文件以恢复到Windows Kerberos缓存。

确保在krb5.conf文件中列出domain_realms。 这是问题的主要根源。

> * 客户端Web浏览器为资源执行HTTP GET。
> * Web服务器返回HTTP 401状态和标头：WWW-Authenticate：Negotiate
> * 客户端生成NegTokenInit，base64对其进行编码，并使用Authorization标头重新提交GET：Authorization：Negotiate \<base64 encoding\>。
> * 服务器解码NegTokenInit，提取支持的MechTypes（在我们的例子中只有Kerberos V5），确保它是预期的一个，然后提取MechToken（Kerberos令牌）并对其进行身份验证。
>   如果需要更多处理，则另一个HTTP 401将返回到客户端，其中WWW-Authenticate头中包含更多数据。 客户端获取信息并生成另一个令牌，将其传递回Authorization标头，直到完成为止。
> * 当客户端经过身份验证后，Web服务器应返回HTTP 200状态，最终的WWW-Authenticate标头和页面内容。

### 4.8.2. GSS / Java Kerberos安装程序

本文档假设您使用的是Windows，但大部分信息也适用于Unix。 org.ietf.jgss类有许多可能的配置参数，主要在krb5.conf / krb5.ini文件中。 有关该格式的更多信息，请访问http://web.mit.edu/kerberos/krb5-1.4/krb5-1.4.1/doc/krb5-admin/krb5.conf.html。

### 4.8.3. login.conf文件

以下配置是在Windows XP中针对IIS和JBoss协商模块运行的基本设置。 系统属性java.security.auth.login.config可用于指向login.conf文件。 login.conf内容可能如下所示：

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

### 4.8.4. krb5.conf / krb5.ini文件

如果未指定，将使用系统默认值。 如果需要，通过将系统属性java.security.krb5.conf设置为指向自定义krb5.conf文件来覆盖。 krb5.conf的内容可能如下所示：

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

### 4.8.5. Windows特定配置

要允许Windows使用当前用户的票证，必须将系统属性javax.security.auth.useSubjectCredsOnly设置为false，并且应添加并正确设置Windows注册表项allowtgtsessionkey，以允许在Kerberos票证授予中发送会话密钥 票。

在Windows Server 2003和Windows 2000 SP4上，这是必需的注册表设置：

```
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Lsa\Kerberos\Parameters
Value Name: allowtgtsessionkey
Value Type: REG_DWORD
Value: 0x01
```

以下是Windows XP SP2上注册表设置的位置：

```
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Lsa\Kerberos\
Value Name: allowtgtsessionkey
Value Type: REG_DWORD
Value: 0x01
```
