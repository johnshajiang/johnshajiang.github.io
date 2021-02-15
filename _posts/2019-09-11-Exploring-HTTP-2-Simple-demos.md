---
layout: post
title: "Exploring HTTP 2: Simple demos"
---

The previous articles in this series have touched the main principles of HTTP/2 protocols. This installment just introduces how to experiment this protocol with real HTTP/2 server and client implementations.

### HTTP/2 implementations
HTTP/2 is already supported by a lot of servers and clients, including Apache httpd 2.4.17+ and Nginx 1.9.5+ on server side and mainstream browsers (e.g. Chrome, FireFox and IE) on client side. In addition, some tools, like Curl and Wireshark, can debug, analyze and visualize this protocol. Please see the [implementation list](https://github.com/http2/http2-spec/wiki/Implementations) and [tool list](https://github.com/http2/http2-spec/wiki/Tools) for more infos.

This article chooses Jetty 9.4.20 as server and Curl 7.65.3 as client for demos. The followings are their version details.

```
$ java -jar start.jar -- version
Jetty Server Classpath:
-----------------------
Version Information on 37 entries in the classpath.
Note: order presented here is how they would appear on the classpath.
      changes to the --module=name command line options will be reflected here.
 0:      1.4.1.v201005082020 | ${jetty.base}/lib/mail/javax.mail.glassfish-1.4.1.v201005082020.jar
 1:                    (dir) | ${jetty.base}/resources
 2:                    3.1.0 | ${jetty.base}/lib/servlet-api-3.1.jar
 3:                 3.1.0.M0 | ${jetty.base}/lib/jetty-schemas-3.1.jar
 4:         9.4.20.v20190813 | ${jetty.base}/lib/jetty-http-9.4.20.v20190813.jar
 5:         9.4.20.v20190813 | ${jetty.base}/lib/jetty-server-9.4.20.v20190813.jar
 6:         9.4.20.v20190813 | ${jetty.base}/lib/jetty-xml-9.4.20.v20190813.jar
 7:         9.4.20.v20190813 | ${jetty.base}/lib/jetty-util-9.4.20.v20190813.jar
 8:         9.4.20.v20190813 | ${jetty.base}/lib/jetty-io-9.4.20.v20190813.jar
 9:         9.4.20.v20190813 | ${jetty.base}/lib/jetty-jndi-9.4.20.v20190813.jar
10:         9.4.20.v20190813 | ${jetty.base}/lib/jetty-security-9.4.20.v20190813.jar
11:                      1.3 | ${jetty.base}/lib/transactions/javax.transaction-api-1.3.jar
12:         9.4.20.v20190813 | ${jetty.base}/lib/jetty-servlet-9.4.20.v20190813.jar
13:         9.4.20.v20190813 | ${jetty.base}/lib/jetty-webapp-9.4.20.v20190813.jar
14:         9.4.20.v20190813 | ${jetty.base}/lib/jetty-plus-9.4.20.v20190813.jar
15:         9.4.20.v20190813 | ${jetty.base}/lib/jetty-annotations-9.4.20.v20190813.jar
16:                      7.1 | ${jetty.base}/lib/annotations/asm-7.1.jar
17:                      7.1 | ${jetty.base}/lib/annotations/asm-analysis-7.1.jar
18:                      7.1 | ${jetty.base}/lib/annotations/asm-commons-7.1.jar
19:                      7.1 | ${jetty.base}/lib/annotations/asm-tree-7.1.jar
20:                      1.3 | ${jetty.base}/lib/annotations/javax.annotation-api-1.3.jar
21:    3.17.0.v20190306-2240 | ${jetty.base}/lib/apache-jsp/org.eclipse.jdt.ecj-3.17.0.jar
22:         9.4.20.v20190813 | ${jetty.base}/lib/apache-jsp/org.eclipse.jetty.apache-jsp-9.4.20.v20190813.jar
23:                   8.5.40 | ${jetty.base}/lib/apache-jsp/org.mortbay.jasper.apache-el-8.5.40.jar
24:                   8.5.40 | ${jetty.base}/lib/apache-jsp/org.mortbay.jasper.apache-jsp-8.5.40.jar
25:                    1.2.5 | ${jetty.base}/lib/apache-jstl/org.apache.taglibs.taglibs-standard-impl-1.2.5.jar
26:                    1.2.5 | ${jetty.base}/lib/apache-jstl/org.apache.taglibs.taglibs-standard-spec-1.2.5.jar
27:         9.4.20.v20190813 | ${jetty.base}/lib/jetty-client-9.4.20.v20190813.jar
28:         9.4.20.v20190813 | ${jetty.base}/lib/jetty-deploy-9.4.20.v20190813.jar
29:                      1.0 | ${jetty.base}/lib/websocket/javax.websocket-api-1.0.jar
30:         9.4.20.v20190813 | ${jetty.base}/lib/websocket/javax-websocket-client-impl-9.4.20.v20190813.jar
31:         9.4.20.v20190813 | ${jetty.base}/lib/websocket/javax-websocket-server-impl-9.4.20.v20190813.jar
32:         9.4.20.v20190813 | ${jetty.base}/lib/websocket/websocket-api-9.4.20.v20190813.jar
33:         9.4.20.v20190813 | ${jetty.base}/lib/websocket/websocket-client-9.4.20.v20190813.jar
34:         9.4.20.v20190813 | ${jetty.base}/lib/websocket/websocket-common-9.4.20.v20190813.jar
35:         9.4.20.v20190813 | ${jetty.base}/lib/websocket/websocket-server-9.4.20.v20190813.jar
36:         9.4.20.v20190813 | ${jetty.base}/lib/websocket/websocket-servlet-9.4.20.v20190813.jar

$ curl -V
curl 7.65.3 (x86_64-apple-darwin17.7.0) libcurl/7.65.3 OpenSSL/1.1.1 zlib/1.2.11 nghttp2/1.39.2
Release-Date: 2019-07-19
Protocols: dict file ftp ftps gopher http https imap imaps ldap ldaps pop3 pop3s rtsp smb smbs smtp smtps telnet tftp 
Features: AsynchDNS HTTP2 HTTPS-proxy IPv6 Largefile libz NTLM NTLM_WB SSL TLS-SRP UnixSockets
```

### Create key store
This article plans to demonstrate the connections on not only h2c (HTTP), but also h2 (HTTPS), so it needs certificates to establish secure connections.

Here just creates a self-signed CA firstly, and then this CA is used to issue a server certificate. The certificates are finally putted into a Java key store.

```
# Generate private key for CA
$ openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:2048 -pkeyopt rsa_keygen_pubexp:65537 -out ca.key

# Generate self-signed CA
$ openssl req -x509 -new -days 3650 -key ca.key -subj "/CN=localhost" -sha256 -out ca.cer

# Generate RSA private key for server
$ openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:2048 -pkeyopt rsa_keygen_pubexp:65537 -out server.key

# Generate certificate signing request for server
$ openssl req -new -key server.key -subj "/CN=localhost" -sha256 -out server.csr

# Generate server certificate based on the above CSR, and sign it with the above CA
$ openssl x509 -req -CAcreateserial -days 3650 -in server.csr -sha256 -CA ca.cer -CAkey ca.key -out server.cer

# Create PKCS12 file
$ cat ca.cer ca.key server.cer server.key > pem
$ openssl pkcs12 -export -nodes -in pem -passout pass:password -out test.pfx
$ openssl pkcs12 -info -in test.pfx -nodes -passin pass:password

# Convert PKCS12 file to Java key store
# Note that the key store password is "password"
$ keytool -importkeystore -srckeystore test.pfx -srcstoretype PKCS12 -srcstorepass "password" -deststoretype PKCS12 -deststorepass "password" -destkeystore test.jks
```

### Create server instance

Create directory `$JETTY_HOME/test-base`, here `$JETTY_HOME` stands for the path to local Jetty distribution.
```
$ cd $JETTY_HOME
$ mkdir test-base
$ cd test-base
```

Create test base with modules supporting HTTP, HTTPS, h2 and h2c.
```
$ java -jar ../start.jar --add-to-start=http,https,http2,http2c,deploy
INFO  : webapp          transitively enabled, ini template available with --add-to-start=webapp
INFO  : server          transitively enabled, ini template available with --add-to-start=server
INFO  : alpn-impl       transitively enabled
INFO  : alpn            transitively enabled, ini template available with --add-to-start=alpn
INFO  : servlet         transitively enabled
INFO  : http2c          initialized in ${jetty.base}/start.ini
INFO  : alpn-impl/alpn-11 dynamic dependency of alpn-impl
INFO  : threadpool      transitively enabled, ini template available with --add-to-start=threadpool
INFO  : ssl             transitively enabled, ini template available with --add-to-start=ssl
INFO  : deploy          initialized in ${jetty.base}/start.ini
INFO  : security        transitively enabled
INFO  : alpn-impl/alpn-9 dynamic dependency of alpn-impl/alpn-11
INFO  : http            initialized in ${jetty.base}/start.ini
INFO  : http2           initialized in ${jetty.base}/start.ini
INFO  : https           initialized in ${jetty.base}/start.ini
INFO  : bytebufferpool  transitively enabled, ini template available with --add-to-start=bytebufferpool
MKDIR : ${jetty.base}/etc
COPY  : ${jetty.home}/modules/ssl/keystore to ${jetty.base}/etc/keystore
MKDIR : ${jetty.base}/webapps
INFO  : Base directory was modified
```

Copy the `test.jks` created in section `Create key store` to test-base/etc directory, and change the file name to `keystore`, which is the default key store name.
```
$ cp /path/to/test.jks $$JETTY_HOME/test-base/etc/keystore
```

Obfuscate key store password.
```
$ java -cp $JETTY_HOME/lib/jetty-util-9.4.20.v20190813.jar org.eclipse.jetty.util.security.Password password
password
OBF:1v2j1uum1xtv1zej1zer1xtn1uvk1v1v
MD5:5f4dcc3b5aa765d61d8327deb882cf99
```

Modify the key store and key manager passwords in `$JETTY_HOME/etc/jetty-ssl-context.xml` with the above obfuscated password.
```
$ vi $JETTY_HOME/etc/jetty-ssl-context.xml
<Set name="KeyStorePassword"><Property name="jetty.sslContext.keyStorePassword" deprecated="jetty.keystore.password" default="OBF:1v2j1uum1xtv1zej1zer1xtn1uvk1v1v"/></Set>
<Set name="KeyManagerPassword"><Property name="jetty.sslContext.keyManagerPassword" deprecated="jetty.keymanager.password" default="OBF:1v2j1uum1xtv1zej1zer1xtn1uvk1v1v"/></Set>
```

### Web application
The structure of this web application, named test.war, is quite simple as the below,
```
$JETTY_HOME/test-base/webapps/test.war
    |-- index
    |-- WEB-INF
        |-- web.xml
```

It contains only one static index file, which contains the below line,
```
HTTP/2 Test
```

The web description file just defines the index as welcome file.
```
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
    metadata-complete="false" version="3.1">

    <welcome-file-list>
        <welcome-file>index</welcome-file>
    </welcome-file-list>
</web-app>
```

Additional `$JETTY_HOME/test-base/webapps/test.xml` is needed for declaring the test web application.
```
<?xml version="1.0"  encoding="ISO-8859-1"?>
<!DOCTYPE Configure PUBLIC "-//Jetty//Configure//EN" "http://www.eclipse.org/jetty/configure_9_0.dtd">

<Configure class="org.eclipse.jetty.webapp.WebAppContext">
  <Set name="contextPath">/</Set>
  <Set name="war"><SystemProperty name="jetty.base" default="."/>/webapps/test.war</Set>
</Configure>
```

### Simple demos
#### Start server
```
$ java -jar $JETTY_HOME/start.jar jetty.http.port=8080 jetty.ssl.port=8443
2019-09-09 11:52:26.750:INFO::main: Logging initialized @210ms to org.eclipse.jetty.util.log.StdErrLog
2019-09-09 11:52:26.839:INFO:oeju.TypeUtil:main: JVM Runtime does not support Modules
2019-09-09 11:52:26.934:WARN:oejx.XmlConfiguration:main: Deprecated method public void org.eclipse.jetty.server.HttpConfiguration.setBlockingTimeout(long) in file:///Users/sha.jiang/work/build/jetty/etc/jetty.xml
2019-09-09 11:52:27.141:INFO:oejs.Server:main: jetty-9.4.20.v20190813; built: 2019-08-13T21:28:18.144Z; git: 84700530e645e812b336747464d6fbbf370c9a20; jvm 1.8.0_221-b78
2019-09-09 11:52:27.164:INFO:oejdp.ScanningAppProvider:main: Deployment monitor [file:///Users/sha.jiang/work/build/jetty/test-base/webapps/] at interval 1
2019-09-09 11:52:27.572:INFO:oejus.SslContextFactory:main: x509=X509@44f75083(1,h=[],w=[]) for Server@2698dc7[provider=null,keyStore=file:///Users/sha.jiang/work/build/jetty/test-base/etc/keystore,trustStore=file:///Users/sha.jiang/work/build/jetty/test-base/etc/keystore]
2019-09-09 11:52:27.675:INFO:oejs.AbstractConnector:main: Started ServerConnector@12405818{SSL,[ssl, alpn, h2, http/1.1]}{0.0.0.0:8443}
2019-09-09 11:52:27.676:INFO:oejs.AbstractConnector:main: Started ServerConnector@9b66ac0{HTTP/1.1,[http/1.1, h2c]}{0.0.0.0:8080}
2019-09-09 11:52:27.677:INFO:oejs.Server:main: Started @1146ms
```
The server started with HTTP port 8080 and HTTPS port 8443.

#### HTTP/2 connection on h2c
```
$ curl -v --http2 http://localhost:8080
*   Trying ::1:8080...
* TCP_NODELAY set
* Connected to localhost (::1) port 8080 (#0)
> GET / HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.65.3
> Accept: */*
> Connection: Upgrade, HTTP2-Settings
> Upgrade: h2c
> HTTP2-Settings: AAMAAABkAARAAAAAAAIAAAAA
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 101 Switching Protocols
* Received 101
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Connection state changed (MAX_CONCURRENT_STREAMS == 1024)!
< HTTP/2 200 
< server: Jetty(9.4.20.v20190813)
< last-modified: Mon, 09 Sep 2019 08:13:06 GMT
< content-length: 12
< accept-ranges: bytes
< 
HTTP/2 Test
* Connection #0 to host localhost left intact
```
Obviousely, the HTTP version is upgraded from HTTP/1.1 to HTTP/2.

#### HTTP/2 connection on h2
```
$ curl -v --http2 --cacert /path/to/ca.cer https://localhost:8443
*   Trying ::1:8443...
* TCP_NODELAY set
* Connected to localhost (::1) port 8443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: ca.cer
  CApath: none
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES256-GCM-SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=localhost
*  start date: Sep  9 03:49:07 2019 GMT
*  expire date: Sep  6 03:49:07 2029 GMT
*  common name: localhost (matched)
*  issuer: CN=localhost
*  SSL certificate verify ok.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x7faa66802000)
> GET / HTTP/2
> Host: localhost:8443
> User-Agent: curl/7.65.3
> Accept: */*
> 
* Connection state changed (MAX_CONCURRENT_STREAMS == 128)!
< HTTP/2 200 
< server: Jetty(9.4.20.v20190813)
< last-modified: Mon, 09 Sep 2019 08:13:06 GMT
< content-length: 12
< accept-ranges: bytes
< 
HTTP/2 Test
* Connection #0 to host localhost left intact
```
This application protocol, namely HTTP in this case, is negotiated to HTTP/2 via ALPN.

