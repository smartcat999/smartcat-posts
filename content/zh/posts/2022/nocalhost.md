---
title: "使用nocalhost调试kubernetes应用"
draft: false
toc: true
description: ""
date: 2022-10-23
featured_image: /images/bg03.jpg
categories: []
tags: [k8s, nocalhost]
---
基于nocalhost本地开发调试云上kubernetes集群应用<!--more-->
#### 1 简介
开源的基于 IDE 的云原生应用开发工具
- 直接在 Kubernetes 集群中构建、测试和调试应用程序
- 提供易于使用的 IDE 插件（支持 VS Code 和 JetBrains），即使在 Kubernetes 集群中进行开发和调试，Nocalhost 也能保持和本地开发一样的开发体验
- 使用即时文件同步进行开发： 即时将您的代码更改同步到远端容器，而无需重建镜像或重新启动容器
#### 2 插件安装
1. 插件商店安装，支持vscode/JetBrains(2021.2及以上)
[插件安装](https://nocalhost.dev/zh-CN/docs/installation)
2. 配置k8s-cluster信息
- 通过添加文件配置 ~/.kube/config
- kubectl config view --minify --raw --flatten 复制内容粘贴添加
[集群配置](https://nocalhost.dev/zh-CN/docs/installation)
#### 3 开发&&调试
##### 3.1 dev mode(replace)
###### 3.1.1 流程
1. 选择workloads，右键Start DevMode
2. 弹窗关联代码仓库，也可以手动关联点击对应工作负载(点击工作负载右键Associate Local DIR选择本地目录关联)，用于同步本地文件夹代码到远程容器
3. 配置dev-config，用于插件远程调试，可以选择在**项目根目录**下面配置.nocalhost/config.yaml
4. 开启Dev Mode

###### 3.1.2 **重点**
- dev-config的配置
    ```yaml
    # .nocalhost/config.yaml
    name: rudder
    serviceType: deployment  # 工作负载的类型
    containers:
      - name: rudder    # 容器名称
        dev:
          gitUrl: ""   # 选择远程仓库的地址用于代码同步
          image: docker.io/2030047311/common-dev:v4  # 远程pod的资源环境
          shell: ""    # 容器使用的shell
          workDir: ""
          storageClass: ""
          resources: null
          persistentVolumeDirs: []
          command:        # 远程调试的命令 对应点击工作负载--右键---remote run/remote debug
            run:
              - chmod +x ./run.sh && ./run.sh
            debug:
              - chmod +x ./debug.sh && ./debug.sh
          debug:
            remoteDebugPort: 9009  # 对应dlv调试的远程端口
          hotReload: false  # 热加载
          sync: null
          env: []
          portForward: []
          patches:  # 拉起dapr的sidecar配置，注意appid以及app-port对应当前应用的配置
            - type: "strategic"
              patch: '{"spec":{"template":{"metadata":{"annotations":{"dapr.io/app-id": "rudder", "dapr.io/app-port": "31234", "dapr.io/enabled": "true","dapr.io/log-as-json": "true" }}}}}'
    ```
    ```shell
    # debug.sh
    #! /bin/sh
    
    export GOPROXY=https://goproxy.cn
    dlv --headless --log --listen :9009 --api-version 2 --accept-multiclient debug github.com/tkeel-io/tkeel/cmd/rudder
    ```
    ```shell
    # run.sh
    #! /bin/sh
    
    export GOPROXY=https://goproxy.cn
    go run github.com/tkeel-io/tkeel/cmd/rudder
    ```
- containers.dev.image 为远程pod的代码执行环境，自定义可能碰到的问题以及解决方案
    ```
    # 官方最新版环境为golang1.16，在调试时可能出现k8s.io相关json库的异常，golang的调试工具dlv支持到1.17，因此选择自定义打包构建
    # nocalhost官方环境: https://github.com/nocalhost/dev-container
    1.在本地开启一个linux amd64系统容器，尽量选择精简版的镜像，影响后续拉取镜像的效率
    2.下载golang的1.17安装包，通过docker cp拷贝到容器里面，设置环境变量
    3.下载dlv的源码包并手动编译可执行文件，也可以使用go install github.com/go-delve/delve/cmd/dlv@master安装，并将生成的dlv文件，通过docker cp拷贝到容器中的/usr/bin目录下
    4.通过docker commit && docker push打包镜像并构建
    # 注意环境还需要依赖git/ssl等模块，选择基础linux环境的时候需要注意
    ```
- k8s的拉取镜像很慢的解决方案
    ```
    # 由于我本地使用的kind的k8s集群，容器运行时默认为containerd，底层为runc.v2,因此使用ctr命令来提前拉取镜像
    $ ctr images pull docker.io/2030047311/common-dev:v1
    # 也可以考虑导入本地镜像，但是需要mac用户需要注意打包的镜像系统版本是否为linux amd64
    # containerd 相关命令
    # ctr -n k8s.io images ls
    # ctr -n k8s.io i import *.tar
    # ctr -n k8s.io i rm ImageID
    ```
- containers.dev.command.run && containers.dev.command.debug && containers.debug.remoteDebugPort
    ```
    # 远程调试用的核心部分，支持直接使用命令go run demo.go或者通过shell去执行
    # 优先选择shell的原因是因为config.yaml的配置修改之后需要重启dev mode才会生效，在shell文件里面修改之后会立刻同步到远端容器文件夹
    # remoteDebugPort 需要和dlv调试工具的port保持一致
    ```
- containers.dev.patches
    ```
    # 用于拉起关联的dapr sidecar，配置需要跟当前代理的上层应用保持一致
    ```
###### 3.1.3 后续
**replace mode会做的工作**
1. 将副本数缩减为
2. 替换容器的镜像为开发镜像
3. 增加一个 sidecar 容器。 为了将本地的源代码改动同步到容器中， Nocalhost 将文件同步服务器运行在一个独立的 sidecar 容器中， 该容器与业务容器挂载相同的同步目录
4. 转发一个本地端口到文件同步服务器，用于文件同步
5. 启动本地文件同步客户端
6. 打开远程终端。 在容器替换成功之后，Nocalhost 会自动打开一个进入到远程容器的终端，通过终端可以直接运行代码

**关闭调试模式之后，可以通过 工作负载-右键-Reset Pod将工作负载重置到之前的状态**
   

#### 4 nocalhost-webui
##### 4.1 install
```shell
1.helm repo add nocalhost "https://nocalhost-helm.pkg.coding.net/nocalhost/nocalhost"
helm repo update

2.helm install nocalhost nocalhost/nocalhost -n nocalhost --create-namespace

3. kubectl -n nocalhost get pods
   NAME                            READY   STATUS    RESTARTS   AGE
   nocalhost-api-b48f7799d-wr4ps   1/1     Running   3          2m7s
   nocalhost-mariadb-0             1/1     Running   0          2m2s
   nocalhost-web-9dd659b8-s89f4    1/1     Running   0          2m7s

4. kubectl -n nocalhost port-forward service/nocalhost-web 8080:80

Email: admin@admin.com
Password: 123456

5.add cluster
```