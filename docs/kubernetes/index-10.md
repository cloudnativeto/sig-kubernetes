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

本文研究了 Route 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。

## APIServerHandler

### Overview

下图为 APIServerHandler 核心组件间关联。主要分为 Restful 和 NonRestful 两部分，director 优先使用 Restful 部分，如果处理成功，则退出，不执行 NonRestful 部分；如果 Restful 部分没有目标功能，则执行 NonRestful 部分。FullHandlerChain 用于 HTTP 处理入口点，链接了中间件功能，并将请求引导至 Director 进行处理。

![routes-api-server-handler-serve.svg](../.gitbook/assets/1%20%282%29.png)

下面为 APIServer 默认的 HandlerChain 构建过程

![image.png](../.gitbook/assets/2%20%282%29.png)

