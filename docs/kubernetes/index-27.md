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

## Procedure

![namespace-controller-main-procedure.svg](../.gitbook/assets/20%20%282%29.png)

NamespaceController 结构体中包含 RateLimitingInterface 示例，当从 Informer 监听到时间变更时，触发的 Event Handler 将事件对象转换为 key 值，并存入 RateLimitingInterface 实例中。

NamespaceController 在启动运行时，会根据要求，启动多个协程，这些协程都执行相同的功能：从 RateLimitingInterface 实例中获取 key 值，并做处理。

## NameResourcesDeleterInterface

### Initialize Unsupported Operations

使用 Discover Client 获取服务端支持的全部 Resource 列表，并根据 Resource 是否需要归属于Namespace 来进行过滤，不需要关联于 Namespace 的资源被过滤。

将获取到的 Resources 进行遍历，根据 GroupVersion 进行分类，并获取该资源支持的操作列表\(Verbs\)，将不支持 **list、deletecollection** 操作的 Resource 记录下来。

![namespace-controller-init-op-cache.svg](../.gitbook/assets/21%20%282%29.png)

### Delete

* [Source](https://sourcegraph.com/github.com/kubernetes/kubernetes/-/blob/pkg/controller/namespace/deletion/namespaced_resources_deleter.go#L95:38)

![namespace-controller-name-resource-deleter-delete.svg](../.gitbook/assets/22%20%282%29.png)

#### Delete All Content

![namespace-controller-delete-all-content.svg](../.gitbook/assets/23%20%282%29.png)

