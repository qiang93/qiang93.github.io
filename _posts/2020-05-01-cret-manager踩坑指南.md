---
layout:     post
title:      cert-manager踩坑指南
subtitle:  	traefik实现自动申请https证书
date:       2020-05-01
author:     huqiang
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
  - traefik
  - kubernetes
  - cert-manager
---

### 前言

说下环境：

K8s: 1.15.8

Cert-manager: 1.14.2

需求：

通过cert-manager管理let' encrypt免费证书，使用dns方式在阿里云域名验证

### 第一个坑

cert-manager自动更新阿里云DNS的插件： https://github.com/pragkent/alidns-webhook

查看alidns-webhook pod日志出现`alidns SDK ErrorCode: InvalidAccessKeyId.NotFound`

明明这个key id和key secret是正确的，折腾的过程就不说了，直接上原因。

直接用`echo 'aaa'|base64`加密会打印末尾的换行符，需要加`-n`参数

`echo -n 'aaa'|base64`

### 第二个坑

能获取到tls.crt、tls.key获取不到ca.crt，暂未找到原因，放弃了。。。