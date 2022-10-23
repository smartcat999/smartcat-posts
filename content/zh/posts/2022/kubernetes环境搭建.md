---
title: "kubernetes和kubesphere部署"
draft: false
toc: true
description: ""
featured_image: /images/kubernetes.png
date: 2022-10-23
categories: []
tags: [k8s]
---
Kubernetes 部署文档<!--more-->
#### 1 docker

##### 1.1 安装依赖

```shell
# docker 
$ sudo apt-get update

$ sudo apt-get install apt-transport-https ca-certificates curl gnupg lsb-release
```

##### 1.2 添加软件源密钥

```shell
# 添加软件源的 GPG 密钥
$ curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
# 官方源
# $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

##### 1.3 添加docker软件源

```shell
# 向 sources.list 中添加 Docker 软件源
$ echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
# 官方源
# $ echo \
#   "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
#   $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

```

##### 1.4 安装docker-ce

```shell
# 更新 apt 软件包缓存，并安装 docker-ce
$ sudo apt-get update
$ sudo apt-get install docker-ce docker-ce-cli containerd.io

```

##### 1.5 更换阿里云镜像源

```shell
# 更换阿里云镜像源
$ sudo mkdir -p /etc/docker
$ sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
      "https://y9wyvozl.mirror.aliyuncs.com",
      "https://hub-mirror.c.163.com",
      "https://mirror.baidubce.com"]
}
EOF
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker

```

#### 2 k3s

##### 2.1 install

```shell
# k3s
$ curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=latest sh -

```

##### 2.2 access

```shell
# cat /etc/rancher/k3s/k3s.yaml

```

##### 2.3 openebs

```shell
# openebs
# helm 安装
# tips: 未安装helm的需要先根据 4.1 安装helm
$ helm repo add openebs https://openebs.github.io/charts
$ helm repo update
$ helm install openebs --namespace openebs openebs/openebs --create-namespace

# k3s 使用helm install异常
# Error: Kubernetes cluster unreachable: Get "http://localhost:8080/version?timeout=32s": dial tcp 127.0.0.1:8080: connect: connection refused
# 解决方式
$ mkdir ~/.kube
$ cp /etc/rancher/k3s/k3s.yaml ~/.kube/
$ export KUBECONFIG=/root/.kube/k3s.yaml


# kubectl 安装
$ kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml
```

#### 3 kubesphere

##### 3.1 install

```shell
# 安装
$ kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.3.0/kubesphere-installer.yaml
   
$ kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.3.0/cluster-configuration.yaml

```

```shell
# 查看安装日志
$ kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l 'app in (ks-install, ks-installer)' -o jsonpath='{.items[0].metadata.name}') -f

```

```shell
# 检查pod安装状态
$ kubectl get pod --all-namespaces
# kubectl get pod -A
$ kubectl get svc/ks-console -n kubesphere-system

```

```shell
# 依赖
# 需要安装
$ apt-get install socat
$ apt-get install conntrack

# 建议安装
$ apt-get install ebtables
$ apt-get install ipset
```

##### 3.2 access

```text
# 确保在安全组中打开了端口 30880，并通过 NodePort (IP:30880) 使用默认帐户和密码 (admin/P@88w0rd) 访问 Web 控制台
url: http://IP:30880

```

```text
# 可插拔组件
https://kubesphere.io/zh/docs/v3.3/pluggable-components/

```

#### 4 helm

##### 4.1 install

```shell
# 直接安装
$ curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

#### 5 kubekey install(k8s && kubesphere)

##### 5.1 install

[all-in-one-on-linux](https://kubesphere.io/zh/docs/v3.3/quick-start/all-in-one-on-linux/)

````shell
# 安装kubekey
$ curl -sfL https://get-kk.kubesphere.io | VERSION=v2.2.1 sh -
$ chmod +x ./kk
$ ./kk create cluster --with-kubesphere v3.3.0

# 验证安装结果
$ kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l 'app in (ks-install, ks-installer)' -o jsonpath='{.items[0].metadata.name}') -f
````

#### 5 uninstall

##### 5.1 kubesphere

[kubesphere-delete.sh](https://raw.githubusercontent.com/kubesphere/ks-installer/release-3.1/scripts/kubesphere-delete.sh)

##### 5.2 k3s

```shell
/usr/local/bin/k3s-uninstall.sh
```
##### 5.3 k8s
```shell
./kk delete cluster
```
##### 6 ingress-nginx


###### 6.1 TLS

1. 生成ca证书以及私钥
    ```shell
    $ openssl req -x509 -sha256 -newkey rsa:4096 -keyout ca.key -out ca.crt -days 356 -nodes -subj "/CN=demo.localdev.me/O=demo.localdev.me"
    ```

2. 生成服务器证书请求文件
    ```shell
    $ openssl req -new -newkey rsa:4096 -keyout server.key \
    -out server.csr -nodes \
    -subj  "/CN=demo.localdev.me/O=demo.localdev.me"  \
    -reqexts SAN -config <(cat /etc/ssl/openssl.cnf \
    <(printf "\n[SAN]\nsubjectAltName=DNS:*.localdev.me"))
    ```

3. CA签署服务器证书
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

4. 查看服务器证书信息
    ```shell
    $ openssl x509 -noout -text -in server.crt
    ```

5. 生成secret
    ```shell
    $ kubectl create secret generic ca-secret --from-file=tls.crt=server.crt --from-file=tls.key=server.key --from-file=ca.crt=ca.crt
    ```
6. ingress配置tls
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
7. 请求测试
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
    # 测试https服务
    root@cluster-001-control-plane:~# curl https://demo.localdev.me:30186 --cacert ca.crt --cert server.crt --key server.key
    <html><body><h1>It works!</h1></body></html>
    ```