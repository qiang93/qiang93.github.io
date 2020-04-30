---
layout:     post
title:      harbor推送镜像unknown blob错误记录
subtitle:  	harbor是一个镜像仓库管理项目
date:       2020-04-28
author:     huqiang
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
    - harbor
	- docker
---

## 前言

只是稍微记录一下这个坑，具体安装细节就不详说了，github上下载的在线安装包通过`sudo ./install.sh`安装的，只是修改了`hostname`和`external_url`并注释了https段落，因为打算在前端用统一的nginx做LB用通配符证书。

## 问题发生

`docker login`登录和WEB UI上都是正常，在`docker push`镜像时发生了`unknown blob`错误。。

经过查看github issue 总算找到了蛛丝马迹。

前端通过nginx做https代理,因为harbor本身已经启动了一个nginx代理，
在最前端的nginx代理配置了以下header,默认处于https模式的AWS ELB也是一样。

```
X-Forwarded-For
X-Forwarded-Proto
```

同时harbor docker-compose启动的nginx也内置了上面的header,需要删除或注释如下header `common/config/nginx/nginx.conf` ,并重启docker-compose容器，否则nginx会重置LB的值，无法正确路由请求。
```
location /v2/ {
  proxy_pass http://core/v2/;
  proxy_set_header Host $http_host;
#      proxy_set_header X-Real-IP $remote_addr;
#      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
#      proxy_set_header X-Forwarded-Proto $scheme;
  proxy_buffering off;
  proxy_request_buffering off;
}
```

事后在docker doc上也发现了这一描述，可以[点此查看](https://docs.docker.com/registry/recipes/nginx/#gotchas)