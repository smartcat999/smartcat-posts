---
title: "ingress-nginx 配置TLS应用实践"
draft: false
toc: true
description: "云原生场景下ingress-nginx配置TLS证书"
date: 2022-10-26
featured_image: /images/nginx.jpeg
categories: []
tags: [k8s, ingress, nginx]
---
在k8s云原生场景下开启ingress-nginx TLS安全配置<!--more-->


##### 1 生成ca证书以及私钥
```shell
$ openssl req -x509 -sha256 -newkey rsa:4096 -keyout ca.key \
  -out ca.crt -days 356 -nodes \
  -subj "/CN=demo.localdev.me/O=demo.localdev.me"
```

##### 2 生成服务器证书请求文件
```shell
$ openssl req -new -newkey rsa:4096 -keyout server.key \
-out server.csr -nodes \
-subj  "/CN=demo.localdev.me/O=demo.localdev.me"  \
-reqexts SAN -config <(cat /etc/ssl/openssl.cnf \
<(printf "\n[SAN]\nsubjectAltName=DNS:*.localdev.me"))
```

##### 3 CA签署服务器证书
```shell
$ openssl x509 -req -sha256 -days 365 \
-in server.csr \
-CA ca.crt \
-CAkey ca.key \
-set_serial 01 \
-out server.crt \
-CAcreateserial \
-extensions SAN \
-extfile <(cat /etc/ssl/openssl.cnf \
<(printf "[SAN]\nsubjectAltName=DNS:*.localdev.me"))
```

##### 4 查看服务器证书信息
```shell
$ openssl x509 -noout -text -in server.crt
```

##### 5 生成secret
```shell
$ kubectl create secret generic ca-secret \
--from-file=tls.crt=server.crt \
--from-file=tls.key=server.key \
--from-file=ca.crt=ca.crt
```
##### 6 ingress配置tls
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "true" # <- 新增
    nginx.ingress.kubernetes.io/auth-tls-secret: default/ca-secret            # <- 新增
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"                  # <- 新增
spec:
  ingressClassName: nginx
  rules:
  - host: demo.localdev.me
    http:
      paths:
      - backend:
          service:
            name: demo
            port:
              number: 80
        path: /
        pathType: Prefix
  tls:                          # <- 新增
  - hosts:                      # <- 新增
    - demo.localdev.me          # <- 新增
    secretName: ca-secret       # <- 新增
```
##### 7 请求测试
```shell
# ingress-svc调整为 NodePort 方式
root@cluster-001-control-plane:~# kubectl get svc ingress-nginx-controller -n ingress-nginx -o wide
NAME                       TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE   SELECTOR
ingress-nginx-controller   NodePort   10.96.117.41   <none>        80:32373/TCP,443:30186/TCP   37h   app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx
```
```shell
#  获取k8s 节点信息node
root@cluster-001-control-plane:~# kubectl get nodes -o wide
NAME                        STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
cluster-001-control-plane   Ready    control-plane   8d    v1.25.2   172.18.0.2    <none>        Ubuntu 22.04.1 LTS   5.4.0-100-generic   containerd://1.6.8


# 编辑hosts新增本地DNS
# 172.18.0.2  demo.localdev.me
```

```shell
# 方法1
# 测试https服务
$ curl https://demo.localdev.me:30186 \
--cacert ca.crt --cert server.crt --key server.key

# output
<html><body><h1>It works!</h1></body></html>


# 方法2
$ openssl s_client -connect demo.localdev.me:30186
```