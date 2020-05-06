---
layout:     post
title:      mattermost学习记录
subtitle:  	mattermost开源slack替代品
date:       2020-05-03
author:     huqiang
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
  - mattermost
  - kubernetes
---

### 前言

---

安装过程就省略了，可以参考官方文档，有多种安装方式`docker`、`helm`等

### 命令行管理

---

通过命令行管理mattermost，可以直接使用`bin/mattermost`，不过官方有意向使用`mmctl`来取代，二者的使用方式可以点下下面两个传送门：

[mattermost](https://docs.mattermost.com/administration/command-line-tools.html#)

[mmctl](https://docs.mattermost.com/administration/mmctl-cli-tool.html#)

这里简单介绍一下`mmctl`吧。`mmctl`是一个远程管理mattermost的工具，不像使用`bin/mattermost`需要下载整个包，通过用户密码登录进行身份验证，所以不需要登录mattermost服务器。

mac下安装非常简单：

> ```bash
> brew install mmctl
> ```

其他系统下安装可以相应[下载安装包](https://github.com/mattermost/mmctl/releases)

首先需要登录连接mattermost服务

```
mmctl auth login https://mattermost.qhtester.site --name local --username admin --password [admin用户密码]
```

验证无误后会出现提示`credentials for "local": "admin@https://mattermost.qhtester.site" stored`

在当前用户目录下会生成一个`.mmctl`的文件存储着相关信息

运行一个命令测试一下吧。

> mmctl config get SqlSettings.DriverName

### 连接外部数据库

通过使用外部mysql数据库，并将config.json配置写入数据库中

官方文档有个坑。。运行`bin/mattermost --config="postgres://mmuser:mostest@localhost:5432/mattermost_test?sslmode=disable\u0026connect_timeout=10"` 官方示例是连接pg数据库，不清楚有没有这个坑，我这边使用的mysql，需要注意[go连接db的格式](https://github.com/go-sql-driver/mysql#examples)

```
mysql://root:asdf1234@tcp(10.98.51.151:3306)/mattermost?charset=utf8mb4,utf8&readTimeout=30s&writeTimeout=30s
```

