---
draft: false
profile: 华北电力大学大四学生。
keywords:
  - Kubernetes
  - APIServer
author: '[杨鼎睿](https://yuque.com/abser)'
date: '2021-05-06T17:04:05.000Z'
title: Kubernetes APIServer Route 架构设计源码阅读
tags:
  - Kubernetes
  - 源码架构图
  - APIServer
avatar: /images/profile/abserari.png
type: post
image: /images/blog/kubernetes-apiserver.png
categories:
  - kubernetes
description: '本文研究了 Route 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。'
---

# API Server Routes

大家好，我是杨鼎睿，这一次给大家带来的是 API Server 的源码阅读。包括之前的 etcd 源码阅读，整个 API Server 共 109 张源码及源码图，文章最后有 API Server 系列目录。欢迎大家的阅读。

本文研究了 Route 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。

## APIServerHandler

### Overview

下图为 APIServerHandler 核心组件间关联。主要分为 Restful 和 NonRestful 两部分，director 优先使用 Restful 部分，如果处理成功，则退出，不执行 NonRestful 部分；如果 Restful 部分没有目标功能，则执行 NonRestful 部分。FullHandlerChain 用于 HTTP 处理入口点，链接了中间件功能，并将请求引导至 Director 进行处理。

![routes-api-server-handler-serve.svg](../.gitbook/assets/1%20%281%29.png)

下面为 APIServer 默认的 HandlerChain 构建过程

![image.png](../.gitbook/assets/2%20%281%29.png)

\[3\] API Server

* [API Server Routes](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-apiserver-route/README.md)
* [API Server API Group](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-apiserver-apigroup/README.md)
* [API Server Storage](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-apiserver-storage/README.md)
* [API Server Cacher](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-apiserver-cacher/README.md)
* [API Server Etcd](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-apiserver-etcd/README.md)
* [API Server Generic API Server](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-apiserver-generic-api-server/README.md)
* [API Server CustomResourceDefinitions](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-apiserver-crd/README.md)
* [API Server Master Server](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-apiserver-master-server/README.md)
* [API Server Aggregator Server](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-apiserver-aggregator-server/README.md)
* [API Server API Server Deprecated \(暂无\)](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-apiserver-route/README.md)

