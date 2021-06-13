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

VersionedResourcesStorageMap saves the mapping of version-&gt;resources-&gt;rest.Storage, the first-level mapping is the version, the second-level is the resource, and the storage is used to solve the creation, modification, and deletion of resource objects.

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

Convert the rest.Storage interface to various operation interfaces, the code is shown below. It can be seen from this that the rest.Storage interface is the key, and we will discuss it in depth later.

![image.png](../.gitbook/assets/9.png)

Take creater as an example. Finally, register creater or namedCreater on the Post method.

![image.png](../.gitbook/assets/10.png)

### Discovery

In the registration code, we can see that when registering the API, available Resources and restful.WebService is returned. Afterward, immediately register the Resources available to the WebService on the root request of the WebService, and the action is GET.

![image.png](../.gitbook/assets/11.png)

