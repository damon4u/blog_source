---
title: logstash-logback-encoder使用
date: 2018-11-19 17:41:50
tags: logback
categories: logback
---

业务场景：
日志系统整体迁入[ELK](https://www.elastic.co/cn/elk-stack)，大体需要两种日志：
* 纯业务日志，记录业务代码中的信息；
* 访问日志，包括请求参数和返回结果。

关于ELK这里不做过多讲解，一般也是公司运维同事维护，这里只介绍如何将日志打印成可被收集输送到es的json格式。

<!-- more -->

选用的工具是目前应用广泛的[Logback JSON encoder](https://github.com/logstash/logstash-logback-encoder)

### 业务日志
首先是logback.xml配置：
```xml
<appender name="JSON_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
		<File>${catalina.base}/logs/stdout-json.log</File>
		<encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder">
				<jsonFactoryDecorator class="com.package.NoEscapingJsonFactoryDecorator"/>
				<!-- 显示行号，类，方法信息 -->
				<includeCallerData>true</includeCallerData>
		</encoder>
		<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
				<fileNamePattern>${catalina.base}/logs/stdout-json.log.%d{yyyy-MM-dd}</fileNamePattern>
		</rollingPolicy>
</appender>
<appender name="JSON_ASYNC" class="ch.qos.logback.classic.AsyncAppender">
		<appender-ref ref="JSON_LOG" />
		<discardingThreshold>0</discardingThreshold>
		<queueSize>1000</queueSize>
		<includeCallerData>true</includeCallerData>
</appender>

<root level="info">
		<appender-ref ref="JSON_ASYNC"/>
</root>
```
其中用到了一个`NoEscapingJsonFactoryDecorator`，用来取消对非ASCII字符的转码，防止换行符导致输出混乱。
```java
public class NoEscapingJsonFactoryDecorator implements JsonFactoryDecorator {
    public NoEscapingJsonFactoryDecorator() {
    }

    public MappingJsonFactory decorate(MappingJsonFactory factory) {
        return (MappingJsonFactory)factory.disable(Feature.ESCAPE_NON_ASCII);
    }
}
```
这里还用到了异步打印的配置，比较常见。
业务代码中打印日志时就正常打印即可，没有什么特别。

这样打印出来的日志就是json格式，默认包含如下字段(手动格式化了)：
```json
{
    "@timestamp": "2018-11-19T18:13:05.167+08:00",
    "@version": 1,
    "message": "your log message",
    "logger_name": "com.package.TestController",
    "thread_name": "http-bio-8181-exec-2",
    "level": "INFO",
    "level_value": 20000,
    "HOSTNAME": "xxx",
    "caller_class_name": "com.package.TestController",
    "caller_method_name": "testMethod",
    "caller_file_name": "TestController.java",
    "caller_line_number": 239
}
```

### 访问日志
其实Logback JSON encoder自身是支持访问信息打印的：[AccessEvent Fields](https://github.com/logstash/logstash-logback-encoder#accessevent-fields)
但是我这里还想打印返回信息，所以没有用这个，还是用[LoggingEvent Fields](https://github.com/logstash/logstash-logback-encoder#loggingevent-fields)，增加自定义字段。
logback.xml配置基本一样：
```xml
<appender name="REQUEST_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
		<File>${catalina.base}/logs/request-json.log</File>
		<encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder">
				<jsonFactoryDecorator class="com.netease.mint.common.core.util.NoEscapingJsonFactoryDecorator"/>
		</encoder>
		<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
				<fileNamePattern>${catalina.base}/logs/request-json.log.%d{yyyy-MM-dd}</fileNamePattern>
		</rollingPolicy>
</appender>

<appender name="REQUEST_ASYNC" class="ch.qos.logback.classic.AsyncAppender">
		<appender-ref ref="REQUEST_LOG" />
		<discardingThreshold>0</discardingThreshold>
		<queueSize>1000</queueSize>
		<includeCallerData>true</includeCallerData>
</appender>

<logger name="mint_request" level="INFO" additivity="false">
		<appender-ref ref="REQUEST_LOG"/>
</logger>
```
重点是在打印日志时，需要设置自定义属性，使用的是[Markers](https://github.com/logstash/logstash-logback-encoder#event-specific-custom-fields)来实现。
文章中介绍了两种技术：
* Markers：只在外层json输出打印，不在message字段中追加；
* StructuredArguments：在外层json输出中打印，同时在message字段中追加。

对于访问日志，只需要在外层json输出打印即可。

文档中提供的例子：
```java
import static net.logstash.logback.marker.Markers.*

/*
 * Add "name":"value" to the JSON output.
 */
logger.info(append("name", "value"), "log message");

/*
 * Add "name1":"value1","name2":"value2" to the JSON output by using multiple markers.
 */
logger.info(append("name1", "value1").and(append("name2", "value2")), "log message");

/*
 * Add "name1":"value1","name2":"value2" to the JSON output by using a map.
 *
 * Note the values can be any object that can be serialized by Jackson's ObjectMapper
 * (e.g. other Maps, JsonNodes, numbers, arrays, etc)
 */
Map myMap = new HashMap();
myMap.put("name1", "value1");
myMap.put("name2", "value2");
logger.info(appendEntries(myMap), "log message");

/*
 * Add "array":[1,2,3] to the JSON output
 */
logger.info(appendArray("array", 1, 2, 3), "log message");

/*
 * Add "array":[1,2,3] to the JSON output by using raw json.
 * This allows you to use your own json seralization routine to construct the json output
 */
logger.info(appendRaw("array", "[1,2,3]"), "log message");

/*
 * Add any object that can be serialized by Jackson's ObjectMapper
 * (e.g. Maps, JsonNodes, numbers, arrays, etc)
 */
logger.info(append("object", myobject), "log message");

/*
 * Add fields of any object that can be unwrapped by Jackson's UnwrappableBeanSerializer.
 * i.e. The fields of an object can be written directly into the json output.
 * This is similar to the @JsonUnwrapped annotation.
 */
logger.info(appendFields(myobject), "log message");
```

我们在实际应用中的方式：
```java
private static final Logger LOGGER = LoggerFactory.getLogger("requestLog");
LOGGER.info(append("requestMethod", request.getMethod())
								.and(append("requestURI", request.getRequestURI()))
								.and(append("queryString", request.getQueryString())
								.and(append("responseStatus", response.getStatus()))
								.and(append("responseStatusReason", HttpStatus.valueOf(status).getReasonPhrase()))
								.and(append("responseContent", logContent))
								.and(append("costTime(ms)", System.currentTimeMillis() - startTime))
				, "log message");
```
这样打印的结果(手动格式化了)如下：
```json
{
    "@timestamp": "2018-11-19T18:23:05.252+08:00",
    "@version": 1,
    "message": "log message",
    "logger_name": "requestLog",
    "thread_name": "http-bio-8181-exec-4",
    "level": "INFO",
    "level_value": 20000,
    "HOSTNAME": "xxx",
    "requestMethod": "GET",
    "requestURI": "/api/test",
    "queryString": "version=-1&groupId=_user_a_",
    "responseStatus": 200,
    "responseStatusReason": "OK",
    "responseContent": "\"-1;-954377400228194665,1721842794179500781,-7660610975189358760;1542623015250\"",
    "costTime(ms)": 4
}
```

完整的打印日志Filter如下：
```java
public class LoggingFilter extends OncePerRequestFilter {
    private static final Logger LOGGER = LoggerFactory.getLogger("requestLog");

    /**
     * 可以打印的返回类型集合
     */
    private static final List<MediaType> VISIBLE_TYPES = Arrays.asList(
            MediaType.valueOf("text/*"),
            MediaType.APPLICATION_FORM_URLENCODED,
            MediaType.APPLICATION_JSON,
            MediaType.APPLICATION_XML,
            MediaType.valueOf("application/*+json"),
            MediaType.valueOf("application/*+xml"),
            MediaType.MULTIPART_FORM_DATA
    );
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        if (isAsyncDispatch(request)) {
            filterChain.doFilter(request, response);
        } else {
            doFilterWrapped(wrapRequest(request), wrapResponse(response), filterChain);
        }
    }

    private void doFilterWrapped(ContentCachingRequestWrapper request, ContentCachingResponseWrapper response, FilterChain filterChain) throws ServletException, IOException {
        String queryString = request.getQueryString();
        long startTime = System.currentTimeMillis();
        try {
            filterChain.doFilter(request, response);
        }
        finally {
            doLog(request, queryString, startTime, response);
            response.copyBodyToResponse();
        }
    }

    private void doLog(ContentCachingRequestWrapper request, String queryString, long startTime, ContentCachingResponseWrapper response) {
        int status = response.getStatus();
        byte[] content = response.getContentAsByteArray();
        String logContent = "";
        if (content.length > 0) {
            logContent = getLogContent(content, response.getContentType(), response.getCharacterEncoding());
        }
        LOGGER.info(append("requestMethod", request.getMethod())
                        .and(append("requestURI", request.getRequestURI()))
                        .and(append("queryString", queryString == null ? "" : queryString))
                        .and(append("responseStatus", status))
                        .and(append("responseStatusReason", HttpStatus.valueOf(status).getReasonPhrase()))
                        .and(append("responseContent", logContent))
                        .and(append("costTime(ms)", System.currentTimeMillis() - startTime))
                , "log message");
    }

    private String getLogContent(byte[] content, String contentType, String contentEncoding) {
        MediaType mediaType = MediaType.valueOf(contentType);
        boolean visible = false;
        for (MediaType me : VISIBLE_TYPES) {
            if (me.includes(mediaType)) {
                visible = true;
                break;
            }
        }
        if (visible) {
            try {
                return new String(content, contentEncoding);
            } catch (UnsupportedEncodingException e) {
                LOGGER.error(e.getMessage());
            }
        }
        return "";
    }

    private static ContentCachingRequestWrapper wrapRequest(HttpServletRequest request) {
        if (request instanceof ContentCachingRequestWrapper) {
            return (ContentCachingRequestWrapper) request;
        } else {
            return new ContentCachingRequestWrapper(request);
        }
    }

    private static ContentCachingResponseWrapper wrapResponse(HttpServletResponse response) {
        if (response instanceof ContentCachingResponseWrapper) {
            return (ContentCachingResponseWrapper) response;
        } else {
            return new ContentCachingResponseWrapper(response);
        }
    }
}
```
