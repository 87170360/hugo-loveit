---
title: "使用Hugo搭建博客"
date: 2023-03-15T12:49:10+08:00
draft: false
tags: ["hugo"]
series: ["hugo blog"]
categories: ["技术"]
---

### 1.系统环境 
CentOS Linux release 8.2.2004 (Core)

### 2.安装Golang
ver>1.18, 用来编译hugo

[Golang安装教程](https://go.dev/doc/install)

安装后设置go proxy

vim ~/.profile
``` shell
# 配置 GOPROXY 环境变量
export GOPROXY=https://proxy.golang.com.cn,direct
# 还可以设置不走 proxy 的私有仓库或组，多个用逗号相隔（可选）
export GOPRIVATE=git.mycompany.com,github.com/my/private

```
source ~/.profile


### 3.安装Hugo

安装gcc
``` shell
yum install -y gcc-c++

```

编译hugo
``` shell
go install -tags extended github.com/gohugoio/hugo@latest
hugo version

```
至此hugo安装完成

