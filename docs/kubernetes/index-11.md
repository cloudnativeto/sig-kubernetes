---
draft: false
profile: 华北电力大学大四学生。
keywords:
  - Kubernetes
  - APIServer
author: '[杨鼎睿](https://yuque.com/abser)'
date: '2021-05-06T17:04:05.000Z'
title: Kubernetes APIServer API Group 架构设计源码阅读
tags:
  - Kubernetes
  - 源码架构图
  - APIServer
avatar: /images/profile/abserari.png
type: post
image: /images/blog/kubernetes-apiserver.png
categories:
  - kubernetes
description: '本文研究了 API Group 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。'
---

# API Server API Group

This paper studies the source code of the  **API Group** section. You should read the source code at the same time. It can enhance your design capacity.

## Resource Management

### Storage

VersionedResourcesStorageMap 保存 Version -&gt; Resource -&gt; rest.Storage 的映射，第一级映射为 Version，二级为 Resource，Storage 用于解决资源对象的创建、更改、删除等操作。

![api-group-vr-storage-map.svg](../.gitbook/assets/3.png)

## Install on API Server

### getResourceNamesForGroup

![api-group-get-resource-name-for-group.svg](../.gitbook/assets/4.png)

### APIGroupVersion

![api-group-api-group-version.svg](../.gitbook/assets/5.png)

### Install APIGroup

![api-group-install-api-group.svg](../.gitbook/assets/6.png)

#### InstallREST

![image.png](../.gitbook/assets/7.png)

### APIInstaller

#### Install

![image.png](../.gitbook/assets/8.png)

#### registerResourceHandlers

将 rest.Storage 接口，转换为各种操作的接口，代码如下所示。从这里可以看出，rest.Storage 接口是关键，后续再深入探讨。

![image.png](../.gitbook/assets/9.png)

以 creater 为例，最终，将 creater 或 namedCreater 注册在 Post 方法上

![image.png](../.gitbook/assets/10.png)

### Discovery

在注册代码中，我们可以看到，注册 API 时，返回了可用的 Resources、restful.WebService。随后，马上将该 WebService 可获取的 Resources 注册在该 WebService 的根请求上，动作为 GET。

![image.png](../.gitbook/assets/11.png)

