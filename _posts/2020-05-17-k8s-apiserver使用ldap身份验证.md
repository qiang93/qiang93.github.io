---
layout:     post
title:      api-server使用ldap身份验证
subtitle:  	api-server身份验证
date:       2020-05-17
author:     huqiang
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
  - mattermost
  - kubernetes
---

***

### 前言

k8s提供了多种身份验证方式

- [**X509 client certificates**](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#x509-client-certs)
- [**Static token file**](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#static-token-file)
- [**Bootstrap tokens**](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#bootstrap-tokens)
- [**Static password file**](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#static-password-file)
- [**Service account tokens**](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#service-account-tokens)
- [**OpenID Connect tokens**](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens)
- [**Webhook token authentication**](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication)
- [**Authenticating proxy**](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#authenticating-proxy)



### 内容

今天要说的是利用Webhook token authentication结合LDAP对私有云K8S apiserver进行身份验证

认证和授权：

- 身份验证：Webhook token authentication
- 授权：RBAC

K8S API访问原理：

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gevtxe9feuj307f046dfo.jpg)

````txt
- Authentication：即身份验证，这个环节它面对的输入是整个http request，它负责对来自client的请求进行身份校验，支持的方法包括：client证书验证（https双向验证）、basic auth、普通token以及jwt token(用于serviceaccount)。APIServer启动时，可以指定一种Authentication方法，也可以指定多种方法。如果指定了多种方法，那么APIServer将会逐个使用这些方法对客户端请求进行验证，只要请求数据通过其中一种方法的验证，APIServer就会认为Authentication成功；在较新版本kubeadm引导启动的k8s集群的apiserver初始配置中，默认支持client证书验证和serviceaccount两种身份验证方式。在这个环节，apiserver会通过client证书或http header中的字段(比如serviceaccount的jwt token)来识别出请求的“用户身份”，包括”user”、”group”等，这些信息将在后面的authorization环节用到。

- Authorization：授权。这个环节面对的输入是http request context中的各种属性，包括：user、group、request path（比如：/api/v1、/healthz、/version等）、request verb(比如：get、list、create等)。APIServer会将这些属性值与事先配置好的访问策略(access policy）相比较。APIServer支持多种authorization mode，包括Node、RBAC、Webhook等。APIServer启动时，可以指定一种authorization mode，也可以指定多种authorization mode，如果是后者，只要Request通过了其中一种mode的授权，那么该环节的最终结果就是授权成功。在较新版本kubeadm引导启动的k8s集群的apiserver初始配置中，authorization-mode的默认配置是”Node,RBAC”。Node授权器主要用于各个node上的kubelet访问apiserver时使用的，其他一般均由RBAC授权器来授权。
````

更详细可参考[这个博客](https://tonybai.com/2018/06/14/the-authentication-and-authorization-of-kubectl-when-accessing-k8s-cluster/)



记录一下细节：

需要事先创建好`.kube/cache`目录，否则在输入用户密码会一直提示输入，这是因为插件没有进行判断

api server需要添加以下配置：

```yaml
    - --authorization-mode=Node,RBAC,Webhook
    - --runtime-config=authentication.k8s.io/v1beta1=true
    - --authentication-token-webhook-config-file=/srv/ldap-webhook-auth.conf
    - --authorization-webhook-config-file=/srv/ldap-webhook-auth.conf
    - --authentication-token-webhook-cache-ttl=30m
    
    volumeMounts:
    - mountPath: /srv/ldap-webhook-auth.conf
      name: webhook-auth
      readOnly: true
      
    volumes:
  	- hostPath:
        path: /srv/ldap-webhook-auth.conf
        type: File
    	name: webhook-auth
```

```yaml
cat /srv/ldap-webhook-auth.conf

apiVersion: v1
kind: Config
clusters:
  - name: kube-ldap
    cluster:
      server: https://kube-ldap.qhtester.site/token

# users refers to the API server's webhook configuration.
users:
  - name: apiserver

current-context: webhook
contexts:
- context:
    cluster: kube-ldap
    user: apiserver
  name: webhook
```

```bash
kubectl create clusterrolebinding user1-admin-binding --clusterrole=cluster-admin --user=user1
```

