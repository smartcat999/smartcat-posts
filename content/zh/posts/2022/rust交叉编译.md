---
title: "Rust交叉编译"
draft: false
toc: true
description: ""
date: 2022-10-29
featured_image: /images/rust-social-wide.jpg
categories: []
tags: [rust, cross-compile]
---
在macos/linux/window下编译Rust程序<!--more-->

##### 1 依赖
###### 1.1 rust依赖

- rustup
- rustc

```shell
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```
[quick-started](https://www.rust-lang.org/learn/get-started)

###### 1.2 编译工具

| target                    | notes                                   |
|---------------------------|-----------------------------------------|
| aarch64-unknown-linux-gnu | ARM64 Linux (kernel 4.1, glibc 2.17+)   |
| i686-pc-windows-gnu       | 32-bit MinGW (Windows 7+)               |
| i686-pc-windows-msvc      | 32-bit MSVC (Windows 7+)                |
| i686-unknown-linux-gnu    | 32-bit Linux (kernel 3.2+, glibc 2.17+) |
| x86_64-apple-darwin       | 64-bit macOS (10.7+, Lion+)             |
| x86_64-pc-windows-gnu     | 64-bit MinGW (Windows 7+)               |
| x86_64-pc-windows-msvc    | 64-bit MSVC (Windows 7+)                |
| x86_64-unknown-linux-gnu  | 64-bit Linux (kernel 3.2+, glibc 2.17+) |
	
更多平台支持见[platform-support](https://doc.rust-lang.org/nightly/rustc/platform-support.html)

##### 2 交叉编译
###### 2.1 macos编译
```shell
$ rustup target list
$ rustup target add aarch64-apple-darwin
$ cargo build --target=aarch64-apple-darwin --release
```
###### 2.2 linux编译
```shell
$ rustup target list
$ rustup target add x86_64-unknown-linux-gnu
$ cargo build --target=x86_64-unknown-linux-gnu --release
```
###### 2.3 windows
```shell
$ rustup target list
$ rustup target add x86_64-pc-windows-gnu
$ apt-get install -y mingw-w64
$ cargo build --target=x86_64-pc-windows-gnu --release
```