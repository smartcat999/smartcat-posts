---
title: "Git 环境配置"
draft: false
toc: true
description: "Git相关配置"
date: 2022-10-23
featured_image: /images/git.png
categories: []
tags: [git]
---
Git版本管理工具配置 <!--more-->

#### 1 本地自建git服务
##### 1.1 环境准备

1. 两台ubuntu虚拟机 + git
2. 安装git
    ```shell
    apt-get update
    apt-get install git -y
    ```

##### 1.2 环境搭建

###### 1.2.1 server端搭建
1. 创建git用户
    ```shell
    useradd git -d /home/git -m -s /bin/bash
    ```
2. 设置账号密码
    ```shell
    passwd git
    # 输入自定义用户密码，此处测试使用 123456
    ```
3. 切换git用户创建git托管服务
    ```shell
    # 切换到git家目录
    su git
    cd ~
    
    # 初始化git仓库
    git init --bare sample.git
    
    # 查看仓库目录
    ls -R
    .:
    sample.git
    
    ./sample.git:
    HEAD  branches  config  description  hooks  info  objects  refs
    ```

###### 1.2.2 client端搭建
1. 上传ssh密钥
    ```shell
    ssh-copy-id -i ~/.ssh/id_rsa.pub git@${serverIP}
    # 输入1.2.1中步骤2设置的git用户密码
    ```
2. clone远程仓库
    ```shell
    git clone git@${serverIP}:/home/git/sample.git
    cd sample
    ```
3. 配置git邮箱/用户名
    ```shell
    git config --global user.email "${邮箱}"
    git config --global user.name "${用户名}"
    ```
4. 改动文件，提交远程仓库
    ```shell
    touch readme.md
    git add .
    git commit -m "feat: init"
    git push
    ```


