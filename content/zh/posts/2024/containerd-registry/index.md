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
containerd配置私有仓库<!--more-->
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





