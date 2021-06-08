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

This paper studies the source code of the **Route** section. You should read the source code at the same time. It can enhance your design capacity.

## APIServerHandler

### Overview

The figure below shows the `APIServerHandler`core assembly. It is mainly divided into Restful and NonRestful two parts.   
- Restful is prioritized, and if the processing is successful, exit. does not execute the NonRestful section;   
- if the RESTful section does not have a target function, the NonRestful section is executed. `FullHandlerChain` is used for HTTP processing entry points, linking the middleware features, and boots the request to the `Director` for processing.

![routes-api-server-handler-serve.svg](../.gitbook/assets/1%20%282%29.png)

The following is the `APIServer`default `HandlerChain` build process.

![image.png](../.gitbook/assets/2%20%282%29.png)

