---
title: "容器安全-镜像改造"
draft: false
toc: true
description: "基于安全基础镜像进行镜像改造"
date: 2023-01-31
featured_image: /images/docker.png
categories: []
tags: [docker, linux, base-image， security]
---
基于安全基础镜像(distroless/alpine/ubi)改在开源组件基础镜像<!--more-->
#### 1 安全基础镜像
1. 开源基础镜像
   - [Google distroless镜像](https://github.com/GoogleContainerTools/distroless)
   - [Redhat ubi镜像](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html/building_running_and_managing_containers/assembly_adding-software-to-a-ubi-container_building-running-and-managing-containers#using-the-ubi-init-images_assembly_adding-software-to-a-ubi-container)
2. 自研基础镜像

#### 2 镜像改造