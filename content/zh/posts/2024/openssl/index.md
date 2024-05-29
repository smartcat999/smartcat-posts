---
title: "openssl证书加密/解密"
draft: false
toc: true
description: "openssl证书加密/解密"
date: 2024-05-29
featured_image: /images/openssl.png
categories: []
tags: [openssl, x509]
---
openssl证书加密/解密<!--more-->
#### 1. PKCS#1 && PKCS#8
> 加密之后格式对比

> PKCS#1
```text
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-256-CBC,F4988B5C2D00B07D554DB0141....

kuMwPlsleSWJglaUt/0dgB1VnRKQFRh9KDYso/zQ6L/fLKYXarp/JpFajTGFHU93
m2jSyUxwxUDo1FYDP4/DxOaTZArziZKZWtWfl15j5SQVB+3ALGHz61v183sFKah3
...

-----END RSA PRIVATE KEY-----
```
> PKCS#8
```text
-----BEGIN ENCRYPTED PRIVATE KEY-----
MIIFLTBXBgkqhkiG9w0BBQ0wSjApBgkqhkiG9w0BBQwwHAQI3TKCC122uPECAggA
MAwGCCqGSIb3DQIJBQAwHQYJYIZIAWUDBAEqBBDLKqqWs+s3FiXLefXT6XA7BIIE
0HviejYBW4ThxHaehv6ZsmRDu9UD2lVg6urcMMZkmPCUJtI2cqKlVBe3niDMlilQ
Vfozyc3fE0TqmO1aXFX41XAM8AHdoIZgOc4pt4lJe75sWjL52OCjyiE643JzeJx+
...
-----END ENCRYPTED PRIVATE KEY-----
```

openssl1.x 和 openssl3.x 加密区别
> openssl1.x 使用 PKCS#1 格式加/解密
```shell
echo test | openssl rsa -in ca.key -out ./ca.key.enc -aes256 -passout stdin
```
> openssl3.x 使用 PKCS#8 格式加/解密，通过 **-traditional** 参数可指定使用 PKCS#1 格式
```shell
echo test | openssl rsa -in ca.key -out ./ca.key.enc -aes256 -passout stdin -traditional
```








