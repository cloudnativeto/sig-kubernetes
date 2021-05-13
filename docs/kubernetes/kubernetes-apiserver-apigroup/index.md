---
title: "Kubernetes APIServer API Group 架构设计源码阅读"
date: 2021-05-07T01:04:05+08:00
draft: false
image: "/images/blog/kubernetes-apiserver.png"
author: "[杨鼎睿](https://yuque.com/abser)"
description: "本文研究了 API Group 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。"
tags: ["Kubernetes","源码架构图", "APIServer"]
categories: ["kubernetes"]
keywords: ["Kubernetes","APIServer"]
type: "post"
avatar: "/images/profile/abserari.png"
profile: "华北电力大学大四学生。"
---

大家好，我是杨鼎睿，这一次给大家带来的是 API Server 的源码阅读。包括之前的 etcd 源码阅读，整个 API Server 共 109 张源码及源码图，文章最后有 API Server 系列目录。欢迎大家的阅读。

本文研究了 API Group 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。

<a name="3AB69"></a>
## Resource Management
<a name="IoPYv"></a>
### Storage
VersionedResourcesStorageMap 保存 Version -> Resource -> rest.Storage 的映射，第一级映射为 Version，二级为 Resource，Storage 用于解决资源对象的创建、更改、删除等操作。

![api-group-vr-storage-map.svg](3.png)
<a name="JaSZh"></a>
## Install on API Server
<a name="kWjUb"></a>
### getResourceNamesForGroup
![api-group-get-resource-name-for-group.svg](4.png)
<a name="Q5bJL"></a>
### APIGroupVersion
![api-group-api-group-version.svg](5.png)
<a name="TWPCZ"></a>
### Install APIGroup
![api-group-install-api-group.svg](6.png)
<a name="EL20h"></a>
#### InstallREST
![image.png](7.png)
<a name="20LJB"></a>
### APIInstaller
<a name="JoeM4"></a>
#### Install
![image.png](8.png)
<a name="QZYZb"></a>
#### registerResourceHandlers
将 rest.Storage 接口，转换为各种操作的接口，代码如下所示。从这里可以看出，rest.Storage 接口是关键，后续再深入探讨。

![image.png](9.png)

以 creater 为例，最终，将 creater 或 namedCreater 注册在 Post 方法上

![image.png](10.png)
<a name="mFGCr"></a>
### Discovery
在注册代码中，我们可以看到，注册 API 时，返回了可用的 Resources、restful.WebService。随后，马上将该 WebService 可获取的 Resources 注册在该 WebService 的根请求上，动作为 GET。

![image.png](11.png)

