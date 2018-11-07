---
title: Filter实现返回结果AES加密
date: 2018-11-07 15:41:50
tags: java
categories: [spring,java]
---

场景：
对于新版本（版本信息在请求header中）接口，使用AES算法对返回结果进行全局加密。
原始接口都是RESTful类型，使用`@ResponseBody`标注，返回json数据。

考虑实现方案：
* web容器过滤器`Filter`
* Spring拦截器`Interceptor`
* Spring的`HttpMessageConverter`
* Spring的`ResponseBodyAdvice`

<!-- more -->

### Spring拦截器Interceptor
开始想到的是使用Spring的拦截器实现，在`postHandle`中对返回结果进行加密。
但是发现拦截器中并不能修改response内容。
想想拦截器的使用场景：
* 日志记录：记录请求信息的日志，以便进行信息监控、信息统计、计算PV（Page View）等；
* 权限检查：如登录检测，进入处理器检测检测是否登录，如果没有直接返回到登录页面；
* 性能监控：有时候系统在某段时间莫名其妙的慢，可以通过拦截器在进入处理器之前记录开始时间，在处理完后记录结束时间，从而得到该请求的处理时间（如果有反向代理，如apache可以自动记录）；
* 通用行为：读取cookie得到用户信息并将用户对象放入请求，从而方便后续流程使用，还有如提取Locale、Theme信息等，只要是多个处理器都需要的即可使用拦截器实现。

我们平时用的最多的是在`preHandle`方法中进行参数校验、权限校验等，并不适用于修改response的场景。
并且Spring文档中也指出：

> Note that postHandle is less useful with `@ResponseBody` and `ResponseEntity` methods for which the response is written and committed within the HandlerAdapter and before postHandle. That means it is too late to make any changes to the response, such as adding an extra header. For such scenarios, you can implement ResponseBodyAdvice and either declare it as an `Controller Advice` bean or configure it directly on `RequestMappingHandlerAdapter`.

返回值response已经在`HandlerAdapter`中提交了，`postHandle`中去修改是没有用的。

### Spring的ResponseBodyAdvice
`ResponseBodyAdvice`这个接口可以在`Controller`处理完请求后，交给`HttpMessageConverter`之前，对返回值进行处理。

```java
/**
 * Invoked after an {@code HttpMessageConverter} is selected and just before
 * its write method is invoked.
 * @param body the body to be written
 * @param returnType the return type of the controller method
 * @param selectedContentType the content type selected through content negotiation
 * @param selectedConverterType the converter type selected to write to the response
 * @param request the current request
 * @param response the current response
 * @return the body that was passed in or a modified, possibly new instance
 */
T beforeBodyWrite(T body, MethodParameter methodParameter, MediaType selectedContentType,
    Class<? extends HttpMessageConverter<?>> selectedConverterType,
    ServerHttpRequest request, ServerHttpResponse response);
```
注意，接口返回的是json数据，那么会使用`MappingJackson2HttpMessageConverter`写到respon body中。
这个方法是在写之前调用的，我们可以将body进行加密，但是，最终还是会以json的形式写入，不符合场景需求。

### Spring的HttpMessageConverter
上面说了，接口返回的是json数据，那么会使用`MappingJackson2HttpMessageConverter`写到respon body中。我们可以重写`MappingJackson2HttpMessageConverter`，在写方法中对数据进行加密：
```java
public class CipherJsonHttpMessageConverter extends MappingJackson2HttpMessageConverter {

    private static final Logger LOGGER = LoggerFactory.getLogger(CipherJsonHttpMessageConverter.class);

    @Override
    protected void writeInternal(Object object, Type type, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
        String resultString = JSONUtil.toJson(object);
        String encryptData = null;
        try {
            encryptData = CipherUtil.encrypt(resultString, "key");
        } catch (Exception e) {
            LOGGER.error("CipherJsonHttpMessageConverter error.", e);
        }
        if (encryptData != null) {
            resultString = encryptData;
        }
        outputMessage.getBody().write(resultString.getBytes(StandardCharsets.UTF_8));
    }
}
```
这样确实能搞定加密。但是这个方法中并不能拿到request，不能获取header中的版本信息。因此，也不符合场景需求。

### 过滤器Filter（选取方案）
其实最开始想到的也是使用过滤器Filter，但是遇到了一些麻烦。
第一件事是，如何在Filter中修改response？
搜索找到答案，实现`HttpServletResponseWrapper`，然后在其中保存response的流数据：
```java
public class BufferedServletResponseWrapper extends HttpServletResponseWrapper {

    private ByteArrayOutputStream output; // 用来保存流数据
    private ServletOutputStream filterOutput;

    public BufferedServletResponseWrapper(HttpServletResponse response) {
        super(response);
        output = new ByteArrayOutputStream();
    }

    /**
     * 巧妙将ServletOutputStream放到公共变量，解决不能多次读写问题
     * @return
     * @throws IOException
     */
    @Override
    public ServletOutputStream getOutputStream() throws IOException {
        if (filterOutput == null) {
            filterOutput = new ServletOutputStream() {
                @Override
                public void write(int b) throws IOException {
                    output.write(b);
                }

                @Override
                public boolean isReady() {
                    return false;
                }

                @Override
                public void setWriteListener(WriteListener writeListener) {
                }
            };
        }
        return filterOutput;
    }

    public byte[] getContentAsByteArray() {
        return output.toByteArray();
    }
}
```
对应的Filter代码：
```java
public class AESEncodeFilter extends OncePerRequestFilter {
    private static final Logger LOGGER = LoggerFactory.getLogger(AESEncodeFilter.class);

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        LOGGER.info("AESEncodeFilter start.");
        if (IPUtil.support(request)) { // 从request的header中检查版本
              BufferedServletResponseWrapper responseWrapper = new BufferedServletResponseWrapper(response);
              filterChain.doFilter(request, responseWrapper);
              String responseStr = new String(responseWrapper.getContentAsByteArray(), response.getCharacterEncoding());
              String encryptResponse = null;
              try {
                  encryptResponse = CipherUtil.encrypt(responseStr, "aesKey");
              } catch (Exception e) {
                  LOGGER.error("AESEncodeFilter failed.", e);
              }
              if (encryptResponse != null) {
                  responseStr = encryptResponse;
              }
              response.getOutputStream().write(responseStr.getBytes());
        } else {
            filterChain.doFilter(request, response);
        }
        LOGGER.info("AESEncodeFilter end.");
    }
}
```
web.xml中的配置：
```xml
<filter>
    <filter-name>AESEncodeFilter</filter-name>
    <filter-class>com.damon4u.filter.filter.AESEncodeFilter</filter-class>
</filter>

<filter-mapping>
    <filter-name>AESEncodeFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```
这样，仅配置一个过滤器，没有问题，加密成功。
但是，业务层还用到了etag过滤器，配合使用时，就出现了奇怪的现象。
第一种配合方式，先配置aes加密过滤器，然后配置etag过滤器：
```xml
<filter>
    <filter-name>AESEncodeFilter</filter-name>
    <filter-class>com.damon4u.filter.filter.AESEncodeFilter</filter-class>
</filter>

<filter-mapping>
    <filter-name>AESEncodeFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>

<!-- ETag -->
<filter>
    <filter-name>etagFilter</filter-name>
    <filter-class>org.springframework.web.filter.ShallowEtagHeaderFilter</filter-class>
    <async-supported>true</async-supported>
</filter>
<filter-mapping>
    <filter-name>etagFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```
我们知道，对于相同的url-pattern，过滤器按照声明顺序执行，返回时，先处理etag，然后处理aes加密。
此时，发现加密成功，日志打印正常，但是response中拿到的数据被截断了，长度与未加密前的数据长度相等。
也就是说aes过滤器处理后，content-length没有更新。

换一种配合方式，先声明etag过滤器，在声明aes过滤器，数据没有被截断。但是，对response添加header操作失败。

debug一下，发现问题出现在`ShallowEtagHeaderFilter`中，它内部使用`ContentCachingResponseWrapper`包装response，对返回结果进行处理。
我们也要对response进行处理，但是content-length没有更新，一定是`BufferedServletResponseWrapper`写法不正确，少实现了一些方法。
阅读`ContentCachingResponseWrapper`代码，发现了一些痕迹，它不光实现了`getOutputStream()`和`getWriter()`两个重要的方法，还是实现了如下两个方法：
```java
@Override
public void flushBuffer() throws IOException {
  // do not flush the underlying response as the content as not been copied to it yet
}

@Override
public void setContentLength(int len) {
  if (len > this.content.size()) {
    this.content.resize(len);
  }
  this.contentLength = len;
}
```
从注释可以看出，重写`flushBuffer()`方法是为了防止在我们的wrapper类复制数据之前，底层的response输出数据到客户端。这样，我们才能在外层修改response，例如添加头信息等。
第二个方法`setContentLength()`是为了更新数据存储区域的，例如，我们加密数据之后，长度变化了，content-length也要变化。

仿照`ContentCachingResponseWrapper`，将重要的方法重写：
```java
public class BufferedServletResponseWrapper extends HttpServletResponseWrapper {

    private final FastByteArrayOutputStream content = new FastByteArrayOutputStream(1024);

    private final ServletOutputStream outputStream = new ResponseServletOutputStream();

    private PrintWriter writer;

    private int statusCode = HttpServletResponse.SC_OK;

    private Integer contentLength;


    /**
     * Create a new ContentCachingResponseWrapper for the given servlet response.
     * @param response the original servlet response
     */
    public BufferedServletResponseWrapper(HttpServletResponse response) {
        super(response);
    }


    @Override
    public void setStatus(int sc) {
        super.setStatus(sc);
        this.statusCode = sc;
    }

    @SuppressWarnings("deprecation")
    @Override
    public void setStatus(int sc, String sm) {
        super.setStatus(sc, sm);
        this.statusCode = sc;
    }

    @Override
    public void sendError(int sc) throws IOException {
        copyBodyToResponse(false);
        try {
            super.sendError(sc);
        }
        catch (IllegalStateException ex) {
            // Possibly on Tomcat when called too late: fall back to silent setStatus
            super.setStatus(sc);
        }
        this.statusCode = sc;
    }

    @Override
    @SuppressWarnings("deprecation")
    public void sendError(int sc, String msg) throws IOException {
        copyBodyToResponse(false);
        try {
            super.sendError(sc, msg);
        }
        catch (IllegalStateException ex) {
            // Possibly on Tomcat when called too late: fall back to silent setStatus
            super.setStatus(sc, msg);
        }
        this.statusCode = sc;
    }

    @Override
    public void sendRedirect(String location) throws IOException {
        copyBodyToResponse(false);
        super.sendRedirect(location);
    }

    @Override
    public ServletOutputStream getOutputStream() throws IOException {
        return this.outputStream;
    }

    @Override
    public PrintWriter getWriter() throws IOException {
        if (this.writer == null) {
            String characterEncoding = getCharacterEncoding();
            this.writer = (characterEncoding != null ? new ResponsePrintWriter(characterEncoding) :
                    new ResponsePrintWriter(WebUtils.DEFAULT_CHARACTER_ENCODING));
        }
        return this.writer;
    }

    /**
     * 重写这个方法才能在外面设置response的header，contentLength等
     * @throws IOException
     */
    @Override
    public void flushBuffer() throws IOException {
        // do not flush the underlying response as the content as not been copied to it yet
    }

    /**
     * 重写这个方法，对response内容进行重写后才能修改response长度
     * @param len
     */
    @Override
    public void setContentLength(int len) {
        if (len > this.content.size()) {
            this.content.resize(len);
        }
        this.contentLength = len;
    }

    // Overrides Servlet 3.1 setContentLengthLong(long) at runtime
    public void setContentLengthLong(long len) {
        if (len > Integer.MAX_VALUE) {
            throw new IllegalArgumentException("Content-Length exceeds ShallowEtagHeaderFilter's maximum (" +
                    Integer.MAX_VALUE + "): " + len);
        }
        int lenInt = (int) len;
        if (lenInt > this.content.size()) {
            this.content.resize(lenInt);
        }
        this.contentLength = lenInt;
    }

    @Override
    public void setBufferSize(int size) {
        if (size > this.content.size()) {
            this.content.resize(size);
        }
    }

    @Override
    public void resetBuffer() {
        this.content.reset();
    }

    @Override
    public void reset() {
        super.reset();
        this.content.reset();
    }

    /**
     * Return the status code as specified on the response.
     */
    public int getStatusCode() {
        return this.statusCode;
    }

    /**
     * Return the cached response content as a byte array.
     */
    public byte[] getContentAsByteArray() {
        return this.content.toByteArray();
    }

    /**
     * Return an {@link InputStream} to the cached content.
     * @since 4.2
     */
    public InputStream getContentInputStream() {
        return this.content.getInputStream();
    }

    /**
     * Return the current size of the cached content.
     * @since 4.2
     */
    public int getContentSize() {
        return this.content.size();
    }

    /**
     * Copy the complete cached body content to the response.
     * @since 4.2
     */
    public void copyBodyToResponse() throws IOException {
        copyBodyToResponse(true);
    }

    /**
     * Copy the cached body content to the response.
     * @param complete whether to set a corresponding content length
     * for the complete cached body content
     * @since 4.2
     */
    protected void copyBodyToResponse(boolean complete) throws IOException {
        if (this.content.size() > 0) {
            HttpServletResponse rawResponse = (HttpServletResponse) getResponse();
            if ((complete || this.contentLength != null) && !rawResponse.isCommitted()) {
                rawResponse.setContentLength(complete ? this.content.size() : this.contentLength);
                this.contentLength = null;
            }
            this.content.writeTo(rawResponse.getOutputStream());
            this.content.reset();
            if (complete) {
                super.flushBuffer();
            }
        }
    }


    private class ResponseServletOutputStream extends ServletOutputStream {

        @Override
        public void write(int b) throws IOException {
            content.write(b);
        }

        @Override
        public void write(byte[] b, int off, int len) throws IOException {
            content.write(b, off, len);
        }

        @Override
        public boolean isReady() {
            return false;
        }

        @Override
        public void setWriteListener(WriteListener writeListener) {

        }
    }


    private class ResponsePrintWriter extends PrintWriter {

        public ResponsePrintWriter(String characterEncoding) throws UnsupportedEncodingException {
            super(new OutputStreamWriter(content, characterEncoding));
        }

        @Override
        public void write(char buf[], int off, int len) {
            super.write(buf, off, len);
            super.flush();
        }

        @Override
        public void write(String s, int off, int len) {
            super.write(s, off, len);
            super.flush();
        }

        @Override
        public void write(int c) {
            super.write(c);
            super.flush();
        }
    }
}
```
这样就可以完美修改response了。
有人可能会说，为什么不直接继承`ContentCachingResponseWrapper`？
主要是因为`ShallowEtagHeaderFilter`中会用一下方法寻找response wrapper，如果我们也用这个类，那么会产生冲突，造成加密无效。
```java
ContentCachingResponseWrapper responseWrapper =
				WebUtils.getNativeResponse(response, ContentCachingResponseWrapper.class);
```

最终的方案是将`ContentCachingResponseWrapper`中的代码考过来，放到一个新的类中，然后在web.xml中注册时，还是先声明etag过滤器，再声明aes加密过滤器。
对于相同内容，AES加密后的结果是相同的，因此不会影响Etag比对。

参考：
[Web on Servlet Stack 1.1.6. Interception](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-handlermapping-interceptor)
