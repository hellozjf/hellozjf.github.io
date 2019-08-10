# 第五章 流畅的API

## 5.1. 易于使用的外观API

从版本4.2开始，HttpClient提供了一个易于使用的Facade API，它基于流畅的界面概念。 Fluent facade API仅公开HttpClient的最基本功能，适用于不需要HttpClient完全灵活性的简单用例。 例如，流畅的外观API使用户不必处理连接管理和资源释放。

以下是通过HC流畅API执行的HTTP请求的几个示例

```
// Execute a GET with timeout settings and return response content as String.
Request.Get("http://somehost/")
        .connectTimeout(1000)
        .socketTimeout(1000)
        .execute().returnContent().asString();
```

```
// Execute a POST with the 'expect-continue' handshake, using HTTP/1.1,
// containing a request body as String and return response content as byte array.
Request.Post("http://somehost/do-stuff")
        .useExpectContinue()
        .version(HttpVersion.HTTP_1_1)
        .bodyString("Important stuff", ContentType.DEFAULT_TEXT)
        .execute().returnContent().asBytes();
```

```
// Execute a POST with a custom header through the proxy containing a request body
// as an HTML form and save the result to the file
Request.Post("http://somehost/some-form")
        .addHeader("X-Custom-header", "stuff")
        .viaProxy(new HttpHost("myproxy", 8080))
        .bodyForm(Form.form().add("username", "vip").add("password", "secret").build())
        .execute().saveContent(new File("result.dump"));
```

也可以直接使用Executor，以便在特定的安全上下文中执行请求，从而对身份验证详细信息进行缓存并重新用于后续请求。

```
Executor executor = Executor.newInstance()
        .auth(new HttpHost("somehost"), "username", "password")
        .auth(new HttpHost("myproxy", 8080), "username", "password")
        .authPreemptive(new HttpHost("myproxy", 8080));
 
executor.execute(Request.Get("http://somehost/"))
        .returnContent().asString();
 
executor.execute(Request.Post("http://somehost/do-stuff")
        .useExpectContinue()
        .bodyString("Important stuff", ContentType.DEFAULT_TEXT))
        .returnContent().asString();
```

### 5.1.1. 响应处理

流畅的外观API通常使用户不必处理连接管理和资源释放。 但是，在大多数情况下，这需要在内存中缓冲响应消息的内容。 强烈建议使用ResponseHandler进行HTTP响应处理，以避免在内存中缓冲内容。

```
Document result = Request.Get("http://somehost/content")
        .execute().handleResponse(new ResponseHandler<Document>() {
 
    public Document handleResponse(final HttpResponse response) throws IOException {
        StatusLine statusLine = response.getStatusLine();
        HttpEntity entity = response.getEntity();
        if (statusLine.getStatusCode() >= 300) {
            throw new HttpResponseException(
                    statusLine.getStatusCode(),
                    statusLine.getReasonPhrase());
        }
        if (entity == null) {
            throw new ClientProtocolException("Response contains no content");
        }
        DocumentBuilderFactory dbfac = DocumentBuilderFactory.newInstance();
        try {
            DocumentBuilder docBuilder = dbfac.newDocumentBuilder();
            ContentType contentType = ContentType.getOrDefault(entity);
            if (!contentType.equals(ContentType.APPLICATION_XML)) {
                throw new ClientProtocolException("Unexpected content type:" +
                    contentType);
            }
            String charset = contentType.getCharset();
            if (charset == null) {
                charset = HTTP.DEFAULT_CONTENT_CHARSET;
            }
            return docBuilder.parse(entity.getContent(), charset);
        } catch (ParserConfigurationException ex) {
            throw new IllegalStateException(ex);
        } catch (SAXException ex) {
            throw new ClientProtocolException("Malformed XML document", ex);
        }
    }
 
    });
```
