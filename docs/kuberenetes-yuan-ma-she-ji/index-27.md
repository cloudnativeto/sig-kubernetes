---
draft: false
profile: 华北电力大学大四学生。
keywords:
  - Kubernetes
  - Controller
author: '[杨鼎睿](https://yuque.com/abser)'
date: '2021-05-10T05:04:05.000Z'
title: Kubernetes Controller Namespace Controller 架构设计源码阅读
tags:
  - Kubernetes
  - 源码架构图
  - Controller
avatar: /images/profile/abserari.png
type: post
image: /images/blog/kubernetes-controller.png
categories:
  - kubernetes
description: '本文研究了 Namespace Controller 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。'
---

# Namespace Controller

大家好，我是杨鼎睿，这一次给大家带来的是 Namespace Controller 部分的源码阅读。 

## Procedure

![namespace-controller-main-procedure.svg](../.gitbook/assets/20%20%281%29.png)

NamespaceController 结构体中包含 RateLimitingInterface 示例，当从 Informer 监听到时间变更时，触发的 Event Handler 将事件对象转换为 key 值，并存入 RateLimitingInterface 实例中。

NamespaceController 在启动运行时，会根据要求，启动多个协程，这些协程都执行相同的功能：从 RateLimitingInterface 实例中获取 key 值，并做处理。

## NameResourcesDeleterInterface

### Initialize Unsupported Operations

使用 Discover Client 获取服务端支持的全部 Resource 列表，并根据 Resource 是否需要归属于Namespace 来进行过滤，不需要关联于 Namespace 的资源被过滤。

将获取到的 Resources 进行遍历，根据 GroupVersion 进行分类，并获取该资源支持的操作列表\(Verbs\)，将不支持 **list、deletecollection** 操作的 Resource 记录下来。

![namespace-controller-init-op-cache.svg](../.gitbook/assets/21%20%281%29.png)

### Delete

![namespace-controller-name-resource-deleter-delete.svg](../.gitbook/assets/22%20%281%29.png)

#### Delete All Content

![namespace-controller-delete-all-content.svg](../.gitbook/assets/23%20%281%29.png)

\[6\] Controllers

* [Controllers Queue](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-controller-queue/README.md)
* [Controllers](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-controller-controllers/README.md)
* [Endporint Controller](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-controller-endpoint-controller/README.md)
* [Namespace Controller](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-controller-namespace-controller/README.md)
* [Node Controllers](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-controller-node-controllers/README.md)

