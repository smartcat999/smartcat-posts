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
