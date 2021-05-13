---
title: "Kubernetes APIServer Aggregator Server 架构设计源码阅读"
date: 2021-05-07T01:04:05+08:00
draft: false
image: "/images/blog/kubernetes-apiserver.png"
author: "[杨鼎睿](https://yuque.com/abser)"
description: "本文研究了 Aggregator Server 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。"
tags: ["Kubernetes","源码架构图", "APIServer"]
categories: ["kubernetes"]
keywords: ["Kubernetes","APIServer"]
type: "post"
avatar: "/images/profile/abserari.png"
profile: "华北电力大学大四学生。"
---

大家好，我是杨鼎睿，这一次给大家带来的是 API Server 的源码阅读。包括之前的 etcd 源码阅读，整个 API Server 共 109 张源码及源码图，文章最后有 API Server 系列目录。欢迎大家的阅读。

本文研究了 Aggregator Server 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。
<a name="SUU8n"></a>
## Service Registration
<a name="JoPpL"></a>
### Workflow
![aggregator-server-workflow.svg](92.png)

通过 Informer 监控 APIService 资源变更，通过 ResourceEventHandler 放入 Controller 队列。Controller 内部处理逻辑与其他 Controller 一致，最终将 APIService 资源变更情况，反映至 Aggregator Server 的 HTTP 处理部分。

<a name="hzb3v"></a>
## Available Condition Controller
<a name="Uo86v"></a>
### Rebuild Service Cache
![aggregator-server-available-service-cache.svg](93.png)

- 监听的是 APIService 资源变更
- 无论是 Add/Update/Delete，重建 cache 方法一致，使用的是从 API Server 获取的服务列表

<a name="lcYd3"></a>
### Change Condition
![aggregator-server-service-condition.svg](94.png)

AvailableConditionController 的运行协程从 queue 中取出内容，并检查该服务状态后，将服务当前上报至 API Server。

[3] API Server
- [API Server Routes](/blog/kubernetes-apiserver-route/)
- [API Server API Group](/blog/kubernetes-apiserver-apigroup/)
- [API Server Storage](/blog/kubernetes-apiserver-storage/)
- [API Server Cacher](/blog/kubernetes-apiserver-cacher/)
- [API Server Etcd](/blog/kubernetes-apiserver-etcd/)
- [API Server Generic API Server](/blog/kubernetes-apiserver-generic-api-server/)
- [API Server CustomResourceDefinitions](/blog/kubernetes-apiserver-crd/)
- [API Server Master Server](/blog/kubernetes-apiserver-master-server/)
- [API Server Aggregator Server](/blog/kubernetes-apiserver-aggregator-server/)
- [API Server API Server Deprecated (暂无)](/blog/kubernetes-apiserver-route/)