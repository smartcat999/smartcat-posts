---
title: "Golang 环境配置"
draft: false
toc: true
description: "Goland环境和版本管理工具配置"
featured_image: /images/golang.webp
categories: []
tags: [golang, g, package]
---
Goland环境和版本管理工具配置 【https://github.com/voidint/g】<!--more-->
#### 1. question

##### 1.1 goland 选择go sdk版本目录提示异常: "The selected directory is not a valid home for Go Sdk"

```
修改glang对应sdk文件 'go1.17.2\src\runtime\internal\sys\zversion.go'
添加const TheVersion = `go1.17.2` 指定版本号
```

##### 1.2 g golang源下载很慢解决方式

- 更换仓库源地址

```
# 设置环境变量
# google.cn
export G_MIRROR=https://golang.google.cn/dl/
# 阿里云
# export G_MIRROR=https://mirrors.aliyun.com/golang/
```

- 更换g本地目录

```
export G_EXPERIMENTAL=true
export G_HOME=~/.g
```
