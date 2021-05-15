---
draft: false
profile: 华北电力大学大四学生。
keywords:
  - Kubernetes
  - APIServer
author: '[杨鼎睿](https://yuque.com/abser)'
date: '2021-05-06T17:04:05.000Z'
title: Kubernetes APIServer Generic API Server 架构设计源码阅读
tags:
  - Kubernetes
  - 源码架构图
  - APIServer
avatar: /images/profile/abserari.png
type: post
image: /images/blog/kubernetes-apiserver.png
categories:
  - kubernetes
description: '本文研究了 Generic API Server 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。'
---

# API Server Generic API Server

本文研究了 Generic API Server 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。 

## Delegation Chain

### Overview

![generic-api-server.svg](../.gitbook/assets/54%20%281%29.png)

### Server Chain

![generic-api-server-server-chain.svg](../.gitbook/assets/55%20%281%29.png)

## HTTP Server

### Handler Chain

HandlerChainBuilderFn 类型定义如下，传入一个 http.Handler 实例，并返回一个 http.Handler 实例，这样，可达到类似中间件的效果。

![image.png](../.gitbook/assets/56%20%281%29.png)

创建 APIServerHandler 时，使用的是下面的方法

![image.png](../.gitbook/assets/57%20%281%29.png)

### Start

最终生成的 preparedAPIAggregator 的启动代码如下，只是简单的调用了 runnable 的 Run 方法，在 Server Chain 中，我们知道，runnable 是 APIAggregator 包含的 GenericAPIServer 生成的 preparedGenericAPIServer 实例。

![image.png](../.gitbook/assets/58%20%281%29.png)

preparedGenericAPIServer 的 Run 方法如下所示

![image.png](../.gitbook/assets/59.png)

