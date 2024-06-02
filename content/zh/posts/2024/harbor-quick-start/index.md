---
title: "harbor helm安装"
draft: false
toc: true
description: "使用helm charts安装harbor"
date: 2024-06-02
featured_image: /images/kubernetes.png
categories: []
tags: [security]
---
在现代软件开发和运维中，容器化和容器编排技术的应用已成为主流。Helm作为Kubernetes的包管理工具，极大地简化了应用的部署和管理。Harbor是一个开源的企业级Docker Registry，提供了安全、合规和高效的镜像管理解决方案。通过结合Helm和Harbor，我们可以轻松地在Kubernetes集群中部署和管理私有镜像仓库。

本文将介绍如何使用Helm安装Harbor。我们将详细讲解从环境准备、Helm安装、Harbor Chart配置到最终的安装和验证过程。<!--more-->
##### 1. 准备工作
**环境要求**
- kubernetes
- helm
> 安装kubernetes，推荐使用 [kubekey](https://github.com/kubesphere/kubekey)
```shell
kk create cluster --with-local-storage
```
##### 2. 配置Harbor Chart
```shell
#添加harbor的helm仓库
helm repo add harbor https://helm.goharbor.io

#下载最新的harbor包
helm pull harbor/harbor --untar
```
```text
[root@ks01 packages]# ll harbor
total 240
-rw-r--r--  1 root root    568 Apr 11 15:56 Chart.yaml
-rw-r--r--  1 root root  11357 Apr 11 15:56 LICENSE
-rw-r--r--  1 root root 187200 Apr 11 15:56 README.md
drwxr-xr-x 14 root root   4096 Jun  2 11:06 templates
-rw-r--r--  1 root root  36791 Jun  2 11:14 values.yaml
```
```shell
# 修改charts包配置，修改https的domain
vim harbor/values.yaml
```
```text
ingress:
  hosts:
    core: dockerhub.kubekey.local
```
```text
# If Harbor is deployed behind the proxy, set it as the URL of proxy
externalURL: https://dockerhub.kubekey.local
```
##### 3. 安装Harbor
```shell
# 安装charts包并创建namespace
helm install harbor -n harbor ./harbor --create-namespace

# 检查pod状态
[root@ks01 packages]# kubectl -n harbor get pod
NAME                                 READY   STATUS    RESTARTS      AGE
harbor-core-6d654dd658-hdxbj         1/1     Running   1 (97m ago)   7h44m
harbor-database-0                    1/1     Running   1 (97m ago)   7h44m
harbor-jobservice-645f8c6469-6lr84   1/1     Running   0             96m
harbor-portal-67df6b8fd4-s2z64       1/1     Running   1 (97m ago)   7h44m
harbor-redis-0                       1/1     Running   1 (97m ago)   7h44m
harbor-registry-7d588b8dc9-hjpcp     2/2     Running   2 (97m ago)   7h44m
harbor-trivy-0                       1/1     Running   1 (97m ago)   7h44m

# 检查ingress
[root@ks01 packages]# kubectl -n harbor get ingress
NAME             CLASS   HOSTS                     ADDRESS   PORTS     AGE
harbor-ingress   nginx   dockerhub.kubekey.local             80, 443   7h45m
```
> 如果集群中未安装 ingress-nginx, 需要额外安装。[参考链接](https://kubernetes.github.io/ingress-nginx/deploy/#quick-start)
```shell
# 安装 ingress-nginx
helm upgrade --install ingress-nginx ingress-nginx   --repo https://kubernetes.github.io/ingress-nginx   --namespace ingress-nginx --create-namespace

# 检查ingressClass
[root@ks01 packages]# kubectl get ingressclasses.networking.k8s.io
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       7h45m
```
> 给 harbor 的ingress 配置 ingressClass
```shell
[root@ks01 packages]# kubectl -n harbor edit ingress harbor-ingress
```
```text
spec:
  ingressClassName: nginx  # <---- 修改为对应 ingressClassName
```
##### 4. 获取 ingress-nginx 网关地址
```shell
# https 服务地址为 https://<节点IP>:<32061>
# http 服务地址为 http://<节点IP>:<30000>
[root@ks01 packages]# kubectl -n ingress-nginx get svc
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.233.11.80    <pending>     80:30000/TCP,443:32061/TCP   7h48m
ingress-nginx-controller-admission   ClusterIP      10.233.36.142   <none>        443/TCP                      7h48m
```
##### 5. 配置TLS
> 导出TLS证书
```shell
kubectl -n harbor get secrets harbor-ingress -o jsonpath={.data.tls\\.key} | base64 -d > /etc/pki/tls/private/dockerhub.kubekey.local.key

kubectl -n harbor get secrets harbor-ingress -o jsonpath={.data.tls\\.crt} | base64 -d > /etc/pki/tls/private/dockerhub.kubekey.local.crt

kubectl -n harbor get secrets harbor-ingress -o jsonpath={.data.ca\\.crt} | base64 -d > /etc/pki/tls/private/dockerhub.kubekey.local.ca
```
> 配置 nginx，需要服务器已安装 nginx。
```text
cat >/etc/nginx/conf.d/harbor.conf<<EOF
upstream backend {
    least_conn;
    server dockerhub.kubekey.local:32061;
}


server {
    listen       443 ssl;
    server_name  dockerhub.kubekey.local;
    ssl_certificate     /etc/pki/tls/private/dockerhub.kubekey.local.crt;
    ssl_certificate_key /etc/pki/tls/private/dockerhub.kubekey.local.key;
    location / {
         proxy_pass https://backend;
         proxy_set_header Host $host;
         proxy_ssl_name $host;
         proxy_ssl_server_name on;
         #proxy_ssl_session_reuse off;
         proxy_ssl_verify on;
         proxy_ssl_trusted_certificate /etc/pki/tls/private/dockerhub.kubekey.local.ca;
    }
}
EOF
```
> 修改 /etc/hosts, 添加 DNS
```text
172.31.19.16  dockerhub.kubekey.local
```
> 更新 nginx 配置
```shell
# 检查 nginx 配置是否正确
[root@ks01 scripts]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

# 更新 nginx 配置
[root@ks01 scripts]# nginx -s reload
```
##### 6. 验证安装
```shell
# 默认密码为admin/Harbor12345
docker login dockerhub.kubekey.local -u admin
Password: Harbor12345
```
> 浏览器访问需要修改本地DNS
```text
# 修改host文件，添加DNS
172.31.19.16  dockerhub.kubekey.local

# 浏览器访问 https://<节点IP>
```