---
layout:     post
title:      haproxy添加let's encrypt证书
subtitle:  	添加let's encrypt免费证书
date:       2020-05-02
author:     huqiang
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
  - haproxy
  - kubernetes
  - let's encrypt
---
### 前言

由于在cert-manager那个坑一直没找到原因，还是直接在LB上添加证书吧

LB是haproxy

### 操作



安装acme.sh脚本

```
curl  https://get.acme.sh | sh
cd ~/.acme.sh/
```

申请证书并配置成haproxy支持的证书

```
# 添加拥有alidns fulll权限的key用户，让其自动更新txt记录
cat >> ~/.bashrc << EOF
export Ali_Key="XXXXXXXX"
export Ali_Secret="XXXXXXXXXXXXXXXX"
EOF
# 指定haproxy安装路径
export DEPLOY_HAPROXY_PEM_PATH=/etc/haproxy
export DEPLOY_HAPROXY_RELOAD="/usr/sbin/service haproxy restart"

acme.sh --issue --dns dns_ali --deploy  -d qhtester.site -d *.qhtester.site --deploy-hook haproxy
```

配置haproxy支持ssl

```
frontend traefik-ssl
  bind 0.0.0.0:443 ssl crt /etc/haproxy/qhtester.site.pem
```

> 注意haproxy后端要指向监听80端口的配置

重启haproxy