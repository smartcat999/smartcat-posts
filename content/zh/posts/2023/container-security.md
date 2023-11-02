---
title: "容器安全-镜像改造"
draft: false
toc: true
description: "基于安全基础镜像进行镜像改造"
date: 2023-01-31
featured_image: /images/docker.png
categories: []
tags: [docker, linux, base-image, security]
---
基于安全基础镜像(distroless/alpine/ubi)改造开源组件基础镜像<!--more-->
#### 1 安全基础镜像
1. 开源基础镜像
   - [Google distroless镜像](https://github.com/GoogleContainerTools/distroless)
   - [Redhat ubi镜像](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html/building_running_and_managing_containers/assembly_adding-software-to-a-ubi-container_building-running-and-managing-containers#using-the-ubi-init-images_assembly_adding-software-to-a-ubi-container)
2. 自研基础镜像

#### 2 镜像改造
##### 2.1 基于alpine的镜像改造
###### 2.1.1 常见问题及解决方式
1. apk 替换软件源
   ```shell
   # Dockerfile 添加
   RUN set -eux && sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories
   ```
2. pip 安装依赖过程中遇到 `Problem with the CMake installation, aborting build. CMake executable is cmake`
   ```shell
   apk --no-cache add cmake
   ```
3. pip 安装依赖过程中遇到 `./bootstrap.sh: line 2: autoreconf: not found`
   ```shell
   apk --no-cache add autoconf
   ```
4. pip 安装依赖过程中遇到 `Can't exec "aclocal": No such file or directory at /usr/share/autoconf/Autom4te/FileUtils.pm line 326`
   ```shell
   apk --no-cache add automake
   ```