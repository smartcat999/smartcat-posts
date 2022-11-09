---
title: "docker定制化基础镜像"
draft: false
toc: true
description: "基于docker进行基础镜像的定制化裁剪"
date: 2022-11-05
featured_image: /images/docker.png
categories: []
tags: [docker, linux, base-image]
---
在保障基础镜像功能的前提下，对镜像进行自定义裁剪，构建安全基础镜像，保障系统的安全性<!--more-->
#### 1. 依赖环境
- docker
- linux系统(ubi/centos/debian/ubuntu)

**Tips: 基础镜像在保证功能的前提下尽量精简尺寸**

#### 2. 构建基础镜像

##### 2.1 构建方式
**构建方式有两种**
1. 通过docker容器构建镜像
2. 通过打包系统文件导入到docker镜像

_方式1: 构建效率快，但是由于docker构建原理(ufs),只能修改、删除上层的镜像文件,导致底层镜像的文件不变会保留在宿主机上_

_方式2: 构建效率慢一些, 因为是直接构建的底层镜像,删除、修改的文件在宿主机上也会删除、修改_

##### 2.2 打包基础镜像
###### 2.2.1 使用物理机、虚拟机、docker安装系统
```shell script
# 为了提高构建效率 本文采用docker获取基础镜像
# 拉取系统镜像
$ docker pull registry.access.redhat.com/ubi8/ubi-minimal:latest

# 运行系统容器
$ docker run --name ubi -d -t registry.access.redhat.com/ubi8/ubi-minimal:latest
```
###### 2.2.2 修改镜像
```shell script
# 进入容器，裁剪镜像
$ docker exec -it ubi bash

# 修改系统配置/安装软件包/删除不需要的依赖
# 根据需求裁剪、更新系统 todo #

# 打包系统需要tar工具库
$ microdnf install tar

# 打包系统，生成打包之后的tar包
$ tar --numeric-owner --exclude=/proc --exclude=/sys --exclude=/usr/share/gdb -cpzvf ubi-beta.tar /
```
###### 2.2.3 导入系统镜像到docker中
```shell script
# 拷贝系统tar包到宿主机/其他服务器
$ docker cp ubi:/ubi-beta.tar .
# 如果是其他服务器可以在宿主机上使用 scp ./ubi-beta.tar ${user}@${host}:/${dir}

# 导入系统镜像到docker中
$ docker import ubi-beta.tar
# output: sha256:828f0c8d274007c7baa7585444172278703ad8400a4e5b9757596ecd5d0af00d

# 修改镜像tag
$ docker tag 828f0c8d2740 registry.access.redhat.com/ubi8/ubi-minimal:v1beta
```
##### 2.3 基础镜像测试

###### 2.3.1 运行镜像测试
```shell script
# 
$ docker run --name ubi-v1beta -d -t registry.access.redhat.com/ubi8/ubi-minimal:v1beta
# 验证被删除的包、新增的包是否存在
```
###### 2.3.2 查看宿主机的镜像文件中是否存在已删除的工具
```shell script
# 如果宿主机的镜像文件中不存在删除的工具文件，说明基础镜像裁剪完成
$ find /var/lib/docker |grep gdb
```