---
layout: post
title: "Exploring HTTP 2: Server push"
---

### HTTP/2 server push
Undoubtedly, [server push](https://tools.ietf.org/html/rfc7540#section-8.2) is one of the famous HTTP/2 features. It allows server to send resources to client even though client doesn't request them. This way would improve performance if client really needs those resources. However, it may waste bandwidth if client doesn't want the additional resources.

This feature looks allow server to send multiple responses to single client request. However, it still applies(pretends) one-response-to-one-request semantic. The additional requests are sent by server itself via [PUSH_PROMISE](https://tools.ietf.org/html/rfc7540#section-6.6) frame. Server should send an additional request(`PUSH_PROMISE`) prior to an additional response. When client receives an request, it can ask server to cancel the push associated to this request via sending [RST_STREAM](https://tools.ietf.org/html/rfc7540#section-6.4).

### Select server and client
Like the previous installment [Simple demos](https://github.com/johnshajiang/blog/wiki/Exploring-HTTP-2:-Simple-demos), this article still uses Jetty as server. However, here uses version 10.0.0 (alpha) due to it implements Servlet 4.0, which supports HTTP/2 server push via standard [PushBuilder](https://javaee.github.io/javaee-spec/javadocs/javax/servlet/http/PushBuilder.html) API.

Because Curl tool doesn't support multiplexing (though libcurl does), so we cannot use it to check the server push. Then, this article doesn't select Curl as client. Fortunately, JDK experimentally provided a HTTP/2-compliant client in JDK 9 (see [JEP 110](https://openjdk.java.net/jeps/110)), and enhanced it in JDK 11 (see [JEP 321](https://openjdk.java.net/jeps/321)). The JDK [HTTP client](https://docs.oracle.com/en/java/javase/11/docs/api/java.net.http/java/net/http/package-summary.html) provides a set of high level APIs to manipulate HTTP, including HTTP/2 server push. Here the HTTP client APIs in JDK 11 are used as client.

### Deploy server
Here just creates a Jetty base to support plain HTTP/2 and deploy web application.

```
$ java -jar ../start.jar --add-to-start=http2c,deploy
INFO  : webapp          transitively enabled, ini template available with --add-to-start=webapp
INFO  : server          transitively enabled, ini template available with --add-to-start=server
INFO  : security        transitively enabled
INFO  : servlet         transitively enabled
INFO  : http2c          initialized in ${jetty.base}/start.ini
INFO  : http            transitively enabled, ini template available with --add-to-start=http
INFO  : threadpool      transitively enabled, ini template available with --add-to-start=threadpool
INFO  : bytebufferpool  transitively enabled, ini template available with --add-to-start=bytebufferpool
INFO  : deploy          initialized in ${jetty.base}/start.ini
MKDIR : ${jetty.base}/webapps
INFO  : Base directory was modified
```

The test web application is still pretty simple, as shown as the below.

```
test.war
    |-- resource
    |-- WEB-INF
        |-- web.xml
        |-- classes
            |-- test
                |-- ServerPushServlet
```
It contains only one Servlet, namely `test.ServerPushServlet`, which pushes a file, namely `resource`. The file contains only one word, exactly `RESOURCE`. The Servlet source is the below,

```
package test;

import java.io.IOException;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.PushBuilder;

public class ServerPushServlet extends HttpServlet {

    private static final long serialVersionUID = -7919477451712778907L;

    protected void doGet(HttpServletRequest request,
            HttpServletResponse response) throws ServletException, IOException {
        PushBuilder pushBuilder = request.newPushBuilder();
        pushBuilder.path("/resource").push();
        response.getWriter().print("Server Push Test");
    }
}
```
Obviously, it's really quite easy to push resources by using `PushBuilder`.

The web description file just declares the above Servlet as usual.

```
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
    metadata-complete="false" version="4.0">

    <servlet>
        <servlet-name>push</servlet-name>
        <servlet-class>test.ServerPushServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>push</servlet-name>
        <url-pattern>/push/*</url-pattern>
    </servlet-mapping>
</web-app>
```

Finally, here starts the server and takes it to listens 8080 as HTTP port.

```
$ java -jar ../start.jar jetty.http.port=8080
2019-09-26 11:21:29.049:INFO::main: Logging initialized @630ms to org.eclipse.jetty.util.log.StdErrLog
2019-09-26 11:21:29.072:INFO::main: Logging initialized @653ms to org.eclipse.jetty.util.log.StdErrLog
2019-09-26 11:21:29.426:INFO:oejs.Server:main: jetty-10.0.0-alpha0; built: 2019-09-12T04:05:56.107Z; git: unknown; jvm 11.0.5+8-LTS
2019-09-26 11:21:29.510:INFO:oejdp.ScanningAppProvider:main: Deployment monitor [file:///path/to/test-base/webapps/] at interval 1
2019-09-26 11:21:29.679:INFO:oejw.StandardDescriptorProcessor:main: NO JSP Support for /test.war, did not find org.eclipse.jetty.jsp.JettyJspServlet
2019-09-26 11:21:29.690:INFO:oejs.session:main: DefaultSessionIdManager workerName=node0
2019-09-26 11:21:29.690:INFO:oejs.session:main: No SessionScavenger set, using defaults
2019-09-26 11:21:29.691:INFO:oejs.session:main: node0 Scavenging every 600000ms
2019-09-26 11:21:29.722:INFO:oejsh.ContextHandler:main: Started o.e.j.w.WebAppContext@465232e9{test.war,/test.war,file:///path/to/test-base/webapps/test.war/,AVAILABLE}{/path/to/test-base/webapps/test.war}
2019-09-26 11:21:29.763:INFO:oejw.StandardDescriptorProcessor:main: NO JSP Support for /, did not find org.eclipse.jetty.jsp.JettyJspServlet
2019-09-26 11:21:29.771:INFO:oejsh.ContextHandler:main: Started o.e.j.w.WebAppContext@7fbdb894{/,file:///path/to/test-base/webapps/test.war/,AVAILABLE}{/path/to/test-base/webapps/test.war}
2019-09-26 11:21:29.803:INFO:oejs.AbstractConnector:main: Started ServerConnector@6c2ed0cd{HTTP/1.1,[http/1.1, h2c]}{0.0.0.0:8080}
2019-09-26 11:21:29.804:INFO:oejs.Server:main: Started @1385ms
```

### Client application
JDK HTTP client APIs provides convenient builder methods to create client (`HttpClient`) and request (`HttpRequest`). And `HttpResponse` provides `BodyHandler` and `PushPromiseHandler` for processing normal and push response bodies respectively.

The below client side application creates a HTTP/2 client to call the Servlet `/push`, and then resolves the multiple response bodies. This application also supports canceling push.

```
import java.io.IOException;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpClient.Version;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.net.http.HttpResponse.BodyHandlers;
import java.net.http.HttpResponse.PushPromiseHandler;
import java.util.Map;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionException;
import java.util.concurrent.ConcurrentHashMap;

public class ServerPushClient {

    public static void main(String[] args) throws Exception {
        // Indicate whether cancel push
        boolean cancelPush = args != null && args.length > 0
                ? Boolean.parseBoolean(args[0])
                : false;

        // Create client supporting HTTP/2
        HttpClient client = HttpClient.newBuilder()
                .version(Version.HTTP_2)
                .build();

        // Create request to call the /push Servlet
        HttpRequest request = HttpRequest.newBuilder()
                .uri(new URI("http://localhost:8080/push"))
                .build();

        Map<HttpRequest, CompletableFuture<HttpResponse<String>>> pushMap
                = new ConcurrentHashMap<>();
        // Handle the push requests and responses
        PushPromiseHandler<String> pushHandler = (initRequest, pushRequest, acceptor) -> {
            CompletableFuture<HttpResponse<String>> pushResponsefuture = null;
            if (!cancelPush) {
                pushResponsefuture = acceptor.apply(BodyHandlers.ofString());
            } else {
                // Directly return a completed CompletableFuture with PushException
                // to indicate to cancel the push
                pushResponsefuture = CompletableFuture.failedFuture(
                        new PushException("Push is cancelled"));
            }
            pushMap.put(pushRequest, pushResponsefuture);
        };

        // Handle the main request and response
        CompletableFuture<HttpResponse<String>> responseFuture
                = client.sendAsync(request, BodyHandlers.ofString(), pushHandler);
        HttpResponse<String> response = responseFuture.join();
        System.out.println("Request: " + response.request());
        System.out.println("Response body: " + response.body());

        // Show the push requests and associated response bodies
        for (HttpRequest pushRequest : pushMap.keySet()) {
            try {
                HttpResponse<String> pushResponse = pushMap.get(pushRequest).join();
                System.out.println("Push Request: " + pushResponse.request());
                System.out.println("Push Response body: " + pushResponse.body());
            } catch (CompletionException e) {
                if (cancelPush && e.getCause() instanceof PushException) {
                    System.out.println("Expected " + e.getCause());
                } else {
                    throw e;
                }
            }
        }
    }
}

class PushException extends IOException {

    private static final long serialVersionUID = -4945636894146587802L;

    public PushException(String message) {
        super(message);
    }

    public PushException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

In addition, JDK HTTP client implementation provides two system properties, namely `jdk.httpclient.HttpClient.log` and `jdk.internal.httpclient.debug`, to enable debug logs. The following test cases just use `-Djdk.httpclient.HttpClient.log=all` for unveiling high level details.

### Receive push
After run the above client side application with the default `cancelPush` flag (exactly `false`), as shown as the below selected output snippet, there are two requests: one is the original client request (`/push`); the other is the push request (`/resource`) created by the server via `PUSH_PROMISE` frame.

```
INFO: FRAME: IN: PUSH_PROMISE: length=158, streamid=1, flags=END_HEADERS  promisedStreamid: 2 headerLength: 154
INFO: REQUEST: PUSH_PROMISE: http://localhost:8080/resource GET
INFO: FRAME: IN: HEADERS: length=113, streamid=1, flags=END_HEADERS
INFO: MISC: handling response (streamid=1)
INFO: HEADERS: RESPONSE HEADERS:
    :status: 200
    date: Fri, 27 Sep 2019 03:37:32 GMT
    expires: Thu, 01 Jan 1970 00:00:00 GMT
    server: Jetty(10.0.0-alpha0)
    set-cookie: JSESSIONID=node0zd0yaz0hmq2k1ly2667xajdup17.node0; Path=/
INFO: RESPONSE: (GET http://localhost:8080/push) 200 HTTP_2 Local port: 49890
INFO: MISC: Reading body on stream 1
Request: http://localhost:8080/push GET
Response body: Server Push Test
INFO: FRAME: IN: HEADERS: length=36, streamid=2, flags=END_HEADERS
INFO: HEADERS: RESPONSE HEADERS (streamid=2):
    :status: 200
    accept-ranges: bytes
    content-length: 8
    date: Fri, 27 Sep 2019 03:37:32 GMT
    last-modified: Thu, 26 Sep 2019 12:00:47 GMT
INFO: RESPONSE: (GET http://localhost:8080/resource) 200 HTTP_2 Local port: 49890
INFO: MISC: Reading body on stream 2
INFO: MISC: Push completed on stream 2 for (GET http://localhost:8080/resource) 200
Push Request: http://localhost:8080/resource GET
Push Response body: RESOURCE
```
Just highlighting that, the push promise is sent in stream 1, however the push response is sent in stream 2, which is reserved by the push promise.

### Cancel push
If run the client application with `cancelPush` flag as `true`, we can get the below lines in the logs.

```
INFO: MISC: No body subscriber for http://localhost:8080/resource GET: Stream 1 cancelled by users handler
INFO: MISC: cancelling stream 2: java.io.IOException: Stream 1 cancelled by users handler
INFO: ERROR: Resetting stream 2 with error code 8
INFO: FRAME: OUT: RESET: length=4, streamid=2, flags=0  Error: Stream cancelled
```
It indicates that the stream 2 is canceled by client via `RST_STREAM` frame, in which the error code is `CANCEL (0x8)`.

