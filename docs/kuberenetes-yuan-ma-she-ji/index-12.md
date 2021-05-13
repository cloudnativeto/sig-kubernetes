---
draft: false
profile: 华北电力大学大四学生。
keywords:
  - Kubernetes
  - APIServer
author: '[杨鼎睿](https://yuque.com/abser)'
date: '2021-05-06T17:04:05.000Z'
title: Kubernetes APIServer Storage 架构设计源码阅读
tags:
  - Kubernetes
  - 源码架构图
  - APIServer
avatar: /images/profile/abserari.png
type: post
image: /images/blog/kubernetes-apiserver.png
categories:
  - kubernetes
description: '本文研究了 Storage 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。'
---

# API Server Storage

大家好，我是杨鼎睿，这一次给大家带来的是 API Server 的源码阅读。包括之前的 etcd 源码阅读，整个 API Server 共 109 张源码及源码图，文章最后有 API Server 系列目录。欢迎大家的阅读。

本文研究了 Storage 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。

## StorageFactory

StorageFactory 的作用是封装并简化对资源的操作。StorageFactory 最主要的作用是根据传入的 GroupResource 获取对应于该资源的存储配置 Config。

![storage-storage-factory.svg](../.gitbook/assets/12%20%281%29.png)

在 API Server 中，StorageFactory 由 StorageFactoryConfig 来生成，StorageFactoryConfig 又通过 EtcdOption 来生成，毕竟无论如何变化，etcd 存储才是最终的目的地。

![image.png](../.gitbook/assets/13%20%281%29.png)

### DefaultStorageFactory

DefaultStorageFactory 是到 1.18 版本前，K8S 内部 StorageFactory 的唯一实现，下面我们来详细分析 DefaultStorageFactory 的模式。

#### Cohabitating Resources

![storage-cohabitating-resources.svg](../.gitbook/assets/14%20%281%29.png)

DefaultStorageFactory 将关联的 GroupResource 组织在一起，从上图可以看到，每一个传入的 GroupResource 是按顺序处理的，因此，关联的 GroupResource 间也是有优先级问题存在的。下图为创建 kube-apiserver 时，使用的 StorageFactory 中关联资源的配置情况

![image.png](../.gitbook/assets/15%20%281%29.png)

### RESTOptionsGetter

Etcd 配置与 StorageFactory 最终都汇入 RESTOptionsGetter 中。RESTOptionsGetter 做为核心配置项，用于通过 GroupResource 找到最终的存储。

![image.png](../.gitbook/assets/16.png)

创建 storage.Interface 的过程如下图所示

![rest-options-getter-landscape.svg](../.gitbook/assets/17.png)

#### StorageFactoryRestOptionFactory

以 StorageFactoryRestOptionFactory 为例，GetRESTOptions 方法步骤如下

* 使用 StorageFactory 生成 Storage Config
* 创建 RESTOptions 结构体，并保存生成的 Storage Config
* 默认使用 generic.UndecoratedStorage 方法作为 Decorator
* 如果开启了 EnableWatchCache 选项，会修改 Decorator

![image.png](../.gitbook/assets/18.png)

#### UndecoratedStorage

UndecoratedStorage 只使用了传入的 storagebackend.Config 参数

![image.png](../.gitbook/assets/19.png)

直接调用 factory.Create 创建后端存储

![image.png](../.gitbook/assets/20.png)

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

