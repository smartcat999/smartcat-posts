---
title: "ingress-nginx 安全配置"
draft: true
toc: true
description: "云原生场景下，设置请求的安全配置"
date: 2022-10-31
featured_image: /images/nginx.jpeg
categories: []
tags: [k8s, ingress, nginx]
---
云原生场景下，使用安全配置加固http请求<!--more-->

##### 1 配置安全响应头

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
    name: custom-headers
    namespace: kubesphere-system
data:
    Content-Security-Policy: Content-Security-Policy: default-src 'self' kubesphere.local; style-src 'self' 'unsafe-inline'; script-src 'self' 'unsafe-inline'; font-src 'self' data:; img-src 'self' data:;
    Strict-Transport-Security: max-age=63072000; includeSubdomains; preload
    X-Content-Type-Options: nosniff
    X-Frame-Options: SAMEORIGIN
    X-XSS-Protection: 1; mode=block
```

##### 2 启用相应头配置 add-headers
```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: ingress-nginx-controller
data:
  access-log-path: /var/log/nginx/access.log
  add-headers: kubesphere-system/custom-headers  # <---- 在nginx的config-map中指定namespaces/configmap
  allow-snippet-annotations: 'true'
  compute-full-forwarded-for: 'true'
  error-log-path: /var/log/nginx/error.log
  force-ssl-redirect: 'true'
  log-format-upstream: >-
    {"time": "$time_iso8601", "remote_addr": "$proxy_protocol_addr",
    "x_forwarded_for": "$proxy_add_x_forwarded_for", "request_id": "$req_id",
    "remote_user": "$remote_user", "bytes_sent": $bytes_sent, "request_time":
    $request_time, "status": $status, "vhost": "$host", "request_proto":
    "$server_protocol", "path": "$uri", "request_query": "$args",
    "request_length": $request_length, "duration": $request_time,"method":
    "$request_method", "http_referrer": "$http_referer", "http_user_agent":
    "$http_user_agent" }
  ssl-redirect: 'true'
  use-forwarded-headers: 'true'
```
