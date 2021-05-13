---
title: "Kubernetes APIServer Generic API Server 架构设计源码阅读"
date: 2021-05-07T01:04:05+08:00
draft: false
image: "/images/blog/kubernetes-apiserver.png"
author: "[杨鼎睿](https://yuque.com/abser)"
description: "本文研究了 Generic API Server 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。"
tags: ["Kubernetes","源码架构图", "APIServer"]
categories: ["kubernetes"]
keywords: ["Kubernetes","APIServer"]
type: "post"
avatar: "/images/profile/abserari.png"
profile: "华北电力大学大四学生。"
---

大家好，我是杨鼎睿，这一次给大家带来的是 API Server 的源码阅读。包括之前的 etcd 源码阅读，整个 API Server 共 109 张源码及源码图，文章最后有 API Server 系列目录。欢迎大家的阅读。

本文研究了 Generic API Server 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。
<a name="plicu"></a>
## Delegation Chain
<a name="P1HWG"></a>
### Overview
![generic-api-server.svg](54.png)
<a name="WUKdL"></a>
### Server Chain
![generic-api-server-server-chain.svg](55.png)
<a name="y7Wgu"></a>
## HTTP Server
<a name="LN3a9"></a>
### Handler Chain
HandlerChainBuilderFn 类型定义如下，传入一个 http.Handler 实例，并返回一个 http.Handler 实例，这样，可达到类似中间件的效果。

![image.png](56.png)

创建 APIServerHandler 时，使用的是下面的方法

![image.png](57.png)
<a name="BZLRs"></a>
### Start
最终生成的 preparedAPIAggregator 的启动代码如下，只是简单的调用了 runnable 的 Run 方法，在 Server Chain 中，我们知道，runnable 是 APIAggregator 包含的 GenericAPIServer 生成的 preparedGenericAPIServer 实例。

![image.png](58.png)

preparedGenericAPIServer 的 Run 方法如下所示

![image.png](59.png)

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