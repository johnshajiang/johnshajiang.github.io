---
layout: post
title: "Exploring HTTP 2: Deploy testing server and client"
---

This just be a note on installing and configuring server and client for HTTP/2 testing.
The system is Ubuntu 18.04.4 LTS. The testing server is Apache httpd 2.4.41, and the testing client is cURL 7.68.0.
OpenSSL 1.1.1d is selected as the underlying SSL/TLS lib.

### Install common tools
```
sudo apt-get install make binutils autoconf automake autotools-dev \
libtool pkg-config zlib1g-dev libcunit1-dev libssl-dev libxml2-dev \
python3-setuptools python3-dev libev-dev libevent-dev libjansson-dev \
libjemalloc-dev g++

sudo apt-get install python3-pip
pip3 install -U cython
```
### Install nghttp2
```
wget https://github.com/nghttp2/nghttp2/releases/download/v1.40.0/nghttp2-1.40.0.tar.gz
tar xvf nghttp2-1.40.0.tar.gz
cd nghttp2-1.40.0
autoreconf -i
automake
autoconf
./configure PYTHON=/usr/bin/python3
make
sudo make install

sudo updatedb
locate libnghttp2.so.14

sudo ln -s /usr/local/lib/libnghttp2.so.14 /lib/x86_64-linux-gnu/libnghttp2.so.14
sudo ln -s /usr/local/lib/libnghttp2.so.14.19.0 /lib/x86_64-linux-gnu/libnghttp2.so.14.19.0
ldconfig
```
### Install OpenSSL
```
wget https://www.openssl.org/source/openssl-1.1.1d.tar.gz
tar xvf openssl-1.1.1d.tar.gz
cd openssl-1.1.1d
./config --prefix=/path/to/http2/openssl-1.1.1d -fPIC no-gost no-shared no-zlib \
enable-ssl3 enable-weak-ssl-ciphers enable-ssl-trace
make
make install
```

### Install Apache dependencies
```
wget http://mirrors.tuna.tsinghua.edu.cn/apache/apr/apr-1.7.0.tar.gz
tar xvf apr-1.7.0.tar.gz
cd apr-1.7.0
./configure --prefix=/path/to/http2/apr-1.7.0
make
make install

wget http://mirrors.tuna.tsinghua.edu.cn/apache/apr/apr-util-1.6.1.tar.gz
tar xvf apr-util-1.6.1.tar.gz
cd apr-util-1.6.1
./configure --prefix=/path/to/http2/apr-util-1.6.1 --with-apr=/path/to/http2/apr-1.7.0
make
make install

wget https://ftp.pcre.org/pub/pcre/pcre-8.44.tar.gz
tar xvf pcre-8.44.tar.gz
cd pcre-8.44
./configure --prefix=/path/to/http2/pcre-8.44
make
make install
```

### Install Apache httpd
```
wget http://mirrors.ibiblio.org/apache/httpd/httpd-2.4.41.tar.gz
tar xvf httpd-2.4.41.tar.gz
cd httpd-2.4.41
./configure --prefix=/path/to/http2/httpd-2.4.41 \
--enable-ssl --enable-http2 --enable-so \
--with-ssl=/path/to/http2/openssl-1.1.1d \
--with-apr=/path/to/http2/apr-1.7.0 \
--with-apr-util=/path/to/http2/apr-util-1.6.1 \
--with-pcre=/path/to/http2/pcre-8.44/bin/pcre-config
```

### Configurations
#### Generate certificates
```
cd /path/to/http2/apache-2.4.41/conf
mkdir ssl
cd ssl

echo "[req]" > openssl.conf
echo "distinguished_name = dn" >> openssl.conf
echo "x509_extensions = ca_ext" >> openssl.conf
echo "[dn]" >> openssl.conf
echo "[ca_ext]" >> openssl.conf
echo "subjectKeyIdentifier = hash" >> openssl.conf
echo "authorityKeyIdentifier = keyid" >> openssl.conf
echo "basicConstraints = critical,CA:TRUE" >> openssl.conf
echo "subjectKeyIdentifier = hash" > v3.ext
echo "authorityKeyIdentifier = keyid,issuer" >> v3.ext

openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:2048 -pkeyopt rsa_keygen_pubexp:65537 -out CA.key
openssl req -config openssl.conf -new -key CA.key -subj "/CN=CA" -sha256 -out CA.csr
openssl x509 -extfile v3.ext -req -CAcreateserial -days 3650 -in CA.csr -sha256 -signkey CA.key -out CA.cer

openssl genpkey -algorithm rsa -pkeyopt rsa_keygen_bits:2048 -pkeyopt rsa_keygen_pubexp:65537 -out SERVER.key
openssl req -new -key EE.key -subj "/CN=<hostname>" -sha256 -out SERVER.csr
openssl x509 -extfile v3.ext -req -CAcreateserial -days 3650 -in EE.csr -sha256 -CA CA.cer -CAkey CA.key -out SERVER.cer
```

#### Configure Apache httpd
```
# Edit conf/extra/httpd-ssl.conf
SSLCertificateFile "conf/ssl/server.cer"
SSLCertificateKeyFile "conf/ssl/server.key"

# Edit conf/httpd.conf
# Enable HTTP/2 protocols
Protocols h2 h2c http/1.1

# Secure (SSL/TLS) connections
Include conf/extra/httpd-ssl.conf

# Enable modules, namely uncomment the below lines
LoadModule socache_shmcb_module modules/mod_socache_shmcb.so
LoadModule http2_module modules/mod_http2.so
LoadModule ssl_module modules/mod_ssl.so
LoadModule auth_digest_module modules/mod_auth_digest.so

ServerName localhost
```

### Install cURL
```
wget https://curl.haxx.se/download/curl-7.68.0.tar.gz
tar xvf curl-7.68.0.tar.gz
cd curl-7.68.0
./configure --prefix=/path/to/http2/curl-7.68.0 --with-ssl=/path/to/http2/openssl-1.1.1d
make
make install
```

### Testing
- Connection via clear HTTP/2 (h2c)

```
curl -v --http2 http://localhost
*   Trying ::1:80...
* TCP_NODELAY set
* Connected to localhost (::1) port 80 (#0)
> GET / HTTP/1.1
> Host: localhost
> User-Agent: curl/7.68.0
> Accept: */*
> Connection: Upgrade, HTTP2-Settings
> Upgrade: h2c
> HTTP2-Settings: AAMAAABkAARAAAAAAAIAAAAA
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 101 Switching Protocols
< Upgrade: h2c
< Connection: Upgrade
* Received 101
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=207
* Connection state changed (MAX_CONCURRENT_STREAMS == 100)!
< HTTP/2 200 
< date: Sun, 00 Jan 1900 00:00:00 GMT
< server: Apache/2.4.41 (Unix) OpenSSL/1.1.1d
< last-modified: Mon, 11 Jun 2007 18:53:14 GMT
< etag: W/"2d-432a5e4a73a80"
< accept-ranges: bytes
< content-length: 45
< content-type: text/html
< 
<html><body><h1>It works!</h1></body></html>
* Connection #0 to host localhost left intact
```

- Connection via encrypted HTTP/2 (h2)

```
curl -vk --http2 https://localhost
*   Trying ::1:443...
* TCP_NODELAY set
* Connected to localhost (::1) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
  CApath: none
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=<hostname>
*  start date: Feb 21 07:36:32 2020 GMT
*  expire date: Feb 18 07:36:32 2030 GMT
*  issuer: CN=CA
*  SSL certificate verify result: unable to get local issuer certificate (20), continuing anyway.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x559e7ed1bbb0)
> GET / HTTP/2
> Host: localhost
> user-agent: curl/7.68.0
> accept: */*
> 
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* old SSL session ID is stale, removing
* Connection state changed (MAX_CONCURRENT_STREAMS == 100)!
< HTTP/2 200 
< date: Fri, 21 Feb 2020 15:03:09 GMT
< server: Apache/2.4.41 (Unix) OpenSSL/1.1.1d
< last-modified: Mon, 11 Jun 2007 18:53:14 GMT
< etag: "2d-432a5e4a73a80"
< accept-ranges: bytes
< content-length: 45
< content-type: text/html
< 
<html><body><h1>It works!</h1></body></html>
* Connection #0 to host localhost left intact
```

