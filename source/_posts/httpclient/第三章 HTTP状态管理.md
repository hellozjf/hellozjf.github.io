# 第三章 HTTP状态管理

最初，HTTP被设计为无状态，面向请求/响应的协议，对于跨越几个逻辑相关的请求/响应交换的有状态会话没有特殊规定。 随着HTTP协议的普及和采用越来越多的系统开始将其用于应用程序，它从未打算用于电子商务应用程序的传输。 因此，对状态管理的支持成为必要。

Netscape Communications当时是Web客户端和服务器软件的领先开发商，它基于专有规范在其产品中实现了对HTTP状态管理的支持。 后来，Netscape尝试通过发布规范草案来标准化该机制。 这些努力促成了通过RFC标准轨道定义的正式规范。 但是，大量应用程序中的状态管理仍然主要基于Netscape草案，并且与官方规范不兼容。 Web浏览器的所有主要开发人员都不得不保持与这些应用程序的兼容性，这极大地促成了标准合规性的碎片化。

## 3.1. HTTP cookie

HTTP cookie是HTTP代理和目标服务器可以交换以维护会话的令牌或短状态信息包。 Netscape的工程师曾经把它称为“神奇的饼干”并且名字卡住了。

HttpClient使用Cookie接口表示抽象cookie令牌。 在其最简单的形式中，HTTP cookie仅仅是名称/值对。 通常，HTTP cookie还包含许多属性，例如有效的域，指定此cookie适用的源服务器上的URL子集的路径，以及cookie有效的最长时间。

SetCookie接口表示由源服务器发送到HTTP代理的Set-Cookie响应头，以便维持会话状态。

ClientCookie接口扩展了Cookie接口，具有其他特定于客户端的功能，例如能够完全检索原始服务器指定的原始cookie属性。 这对于生成Cookie标头很重要，因为某些Cookie规范要求Cookie标头只有在Set-Cookie标头中指定时才应包含某些属性。

以下是创建客户端cookie对象的示例：

```
BasicClientCookie cookie = new BasicClientCookie("name", "value");
// Set effective domain and path attributes
cookie.setDomain(".mycompany.com");
cookie.setPath("/");
// Set attributes exactly as sent by the server
cookie.setAttribute(ClientCookie.PATH_ATTR, "/");
cookie.setAttribute(ClientCookie.DOMAIN_ATTR, ".mycompany.com");
```

## 3.2. Cookie规范

CookieSpec接口表示cookie管理规范。 cookie管理规范有望强制执行：

* 解析Set-Cookie标头的规则。
* 解析cookie的验证规则。
* 格式化给定主机，端口和原始路径的Cookie标头。

HttpClient附带了几个CookieSpec实现：

* 标准严格：符合RFC 6265第4节定义的良好行为配置文件的语法和语义的状态管理策略。
* 标准：符合RFC 6265定义的更宽松的配置文件的状态管理策略，第4节旨在与不符合良好行为的配置文件的现有服务器进行互操作。
* Netscape草案（已废弃）：此政策符合Netscape Communications发布的原始草案规范。 除非绝对有必要与遗留代码兼容，否则应该避免使用它。
* RFC 2965（已废弃）：符合RFC 2965定义的过时状态管理规范的状态管理策略。请勿在新应用程序中使用。
* RFC 2109（已废弃）：符合RFC 2109定义的过时状态管理规范的状态管理策略。请勿在新应用程序中使用。
* 浏览器兼容性（已废弃）：此策略致力于严密模仿旧版浏览器应用程序（如Microsoft Internet Explorer和Mozilla FireFox）的（错误）行为。 请不要在新的应用程序中使用。
* 默认值：默认cookie策略是一种综合策略，它根据与HTTP响应一起发送的cookie的属性（例如版本属性，现在已过时）选择RFC 2965，RFC 2109或Netscape草案兼容实现。 在HttpClient的下一个次要版本中，将弃用此策略以支持标准（符合RFC 6265）实现。
* 忽略cookie：忽略所有cookie。

强烈建议在新应用程序中使用标准或标准严格策略。 应使用过时的规范与旧系统兼容。 在HttpClient的下一个主要版本中将删除对过时规范的支持。

## 3.3. 选择cookie策略

如果需要，可以在HTTP客户端设置Cookie策略，并在HTTP请求级别覆盖。

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

## 3.4. 自定义cookie策略

为了实现自定义cookie策略，应该创建CookieSpec接口的自定义实现，创建CookieSpecProvider实现以创建和初始化自定义规范的实例，并使用HttpClient注册工厂。 一旦注册了自定义规范，就可以像标准cookie规范一样激活它。

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

## 3.5. Cookie持久性

HttpClient可以与实现CookieStore接口的持久性cookie存储的任何物理表示一起使用。 名为BasicCookieStore的默认CookieStore实现是一个由java.util.ArrayList支持的简单实现。 当容器对象被垃圾收集时，存储在BasicClientCookie对象中的Cookie将丢失。 如有必要，用户可以提供更复杂的实现。

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

## 3.6. HTTP状态管理和执行上下文

在HTTP请求执行过程中，HttpClient将以下与状态管理相关的对象添加到执行上下文中：

* 查找实例表示实际的cookie规范注册表。 在本地上下文中设置的此属性的值优先于默认值。
* CookieSpec实例表示实际的cookie规范。
* CookieOrigin实例表示原始服务器的实际详细信息。
* CookieStore实例表示实际的cookie存储。 在本地上下文中设置的此属性的值优先于默认值。

本地HttpContext对象可用于在请求执行之前自定义HTTP状态管理上下文，或在请求执行后检查其状态。 还可以使用单独的执行上下文来实现每个用户（或每个线程）状态管理。 在本地上下文中定义的cookie规范注册表和cookie存储将优先于在HTTP客户端级别设置的默认值

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
