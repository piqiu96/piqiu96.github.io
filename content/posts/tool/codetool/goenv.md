---
title: "Go环境搭建"
date: 2021-12-28T15:13:44+08:00
draft: false
categories: ["Golang"]
---

## 一、下载地址

https://golang.google.cn/dl/

## 二、环境变量

```shell
// 设置环境变量内容
export GOHOME=/Users/qiuyong/Develop/dev_env/goenv
export GOROOT=$GOHOME/go
export GOPATH=$GOHOME/gopath
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin

// 生效环境变量
source /home/work/.bash_profile

// 检验
go env --json
```

## 三、配置环境

```shell
go env -w GO111MODULE="on"                          ## 开启go mod模式
go env -w GONOPROXY=\*\*.xxx.com\*\*                ## 配置GONOPROXY环境变量，所有百度内代码，不走代理
go env -w GONOSUMDB=\*                              ## 配置GONOSUMDB，目前有一些代码库还不支持sumdb索引，暂时屏蔽此功能
go env -w GOPROXY=https://goproxy.xxx,direct        ## 配置GOPROXY
go env -w GOPRIVATE=\*.xxx.com
```

