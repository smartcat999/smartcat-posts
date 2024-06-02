---
title: "containerd registry configuration"
draft: false
toc: true
description: "containerd 配置私有仓库"
date: 2024-05-28
featured_image: /images/docker.png
categories: []
tags: [containerd, docker, k8s]
---
在容器化技术的普及过程中，容器镜像的管理和分发变得至关重要。作为容器运行时的核心组件，containerd广泛应用于各种容器管理平台，如Kubernetes、Docker等。为了确保镜像的安全性和访问的可靠性，配置私有镜像仓库已成为企业部署容器化应用的常见实践。

本文将介绍如何为containerd配置私有镜像仓库。我们将详细讲解从基础环境准备、私有镜像仓库的搭建、containerd配置文件的修改到最终的验证过程。<!--more-->
#### 1 配置文件
**注意containerd版本，当前配置仅适用于 containerd v2.0 以下版本的**

**containerd v2.0 以上的版本请参考本文末尾的链接**

**/etc/containerd/config.toml**

#### 2 启用 cri 插件
```text
# vim /etc/containerd/config.toml

#注释 disabled_plugins = ["cri"]
# disabled_plugins = ["cri"]
```

#### 3 配置 mirrors

```text
[plugins."io.containerd.grpc.v1.cri".registry]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."dockerhub.kubekey.local"]
      endpoint = ["http://dockerhub.kubekey.local:80"]
```

#### 4 配置http
```text
[plugins."io.containerd.grpc.v1.cri".registry]
  [plugins."io.containerd.grpc.v1.cri".registry.configs]
    [plugins."io.containerd.grpc.v1.cri".registry.configs."dockerhub.kubekey.local".tls]
      insecure_skip_verify = true
```

#### 5 配置仓库认证信息
```text
[plugins."io.containerd.grpc.v1.cri".registry]
    [plugins."io.containerd.grpc.v1.cri".registry.configs."dockerhub.kubekey.local:80".auth]
      username = "xxx"
      password = "xxx"
      auth = ""
      identitytoken = ""
```

#### 6 重启 containerd
```shell
systemctl restart containerd

crictl pull dockerhub.kubekey.local/huawei/alpine:edge
```


**参考链接🔗**：[cri-registry](https://github.com/containerd/containerd/blob/main/docs/cri/registry.md)





