---
draft: false
profile: 华北电力大学大四学生。
keywords:
  - Kubernetes
  - APIServer
author: '[杨鼎睿](https://yuque.com/abser)'
date: '2021-05-06T17:04:05.000Z'
title: Kubernetes APIServer Master Server 架构设计源码阅读
tags:
  - Kubernetes
  - 源码架构图
  - APIServer
avatar: /images/profile/abserari.png
type: post
image: /images/blog/kubernetes-apiserver.png
categories:
  - kubernetes
description: '本文研究了 Master Server 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。'
---

# API Server Master Server

This article has studied the source code of the Master Server part, equipped with the source code for further understanding, which can deepen the understanding and enhance related design capabilities.

## Server Handler

### Serve HTTP Procedure

![master-server-server-handler-overview.svg](../.gitbook/assets/76%20%281%29.png)

### Add Route

![image.png](../.gitbook/assets/77%20%281%29.png)

![image.png](../.gitbook/assets/78%20%281%29.png)

## Resource Handler

### Install Legacy Resource

First, determine whether the resource configuration of the v1 version is enabled. If enabled, the corresponding resource processing API will be installed. Note that the two core components, StorageFactory and RESTOptionsGetter, have been explained in more detail before.

Create a LegacyRESTStorageProvider object, save the StorageFactory and other necessary information, and then pass in the method InstallLegacyAPI, along with RESTOptionsGetter.

![image.png](../.gitbook/assets/79.png)

InstallLegacyAPI uses the passed parameters to create APIGroupInfo and install it.

![image.png](../.gitbook/assets/80%20%281%29.png)

#### NewLegacyRESTStorage

* Create APIGroupInfo.

![image.png](../.gitbook/assets/81%20%281%29.png)

* Create various types of RESTStorage, not all of them are listed in the figure below.

![image.png](../.gitbook/assets/82%20%281%29.png)

* Build resources for Storage mapping.

![image.png](../.gitbook/assets/83%20%281%29.png)

* An associate resource to Storage mapping on version v1.

![image.png](../.gitbook/assets/84%20%281%29.png)

#### REST

Each resource type has its own REST package. Generally speaking, REST only needs to simply encapsulate a Store. When creating, it will register **NewFunc**, **NewListFunc**, and **behavior strategies** that match the resource type.

![image.png](../.gitbook/assets/85%20%281%29.png)

Note that REST is not necessarily only one Store, such as Posstorage.

![image.png](../.gitbook/assets/86%20%281%29.png)

### RESTStorageProvider

![master-server-rest-storage-provider.svg](../.gitbook/assets/87%20%281%29.png)

RESTStorageProvider cooperates with Resource Config and REST Options to create APIGroupInfo, which is used to register resource processing methods with API Server.

RESTOptionsGetter registers the Store with the Storage Map according to the version and resource check method in APIResourceConfigSource, and finally mounts the Storage Map to APIGroupInfo. Take Auto Scaling as an example, the code is as follows.

![image.png](../.gitbook/assets/88%20%281%29.png)

The code to create the v1 version of Storage is as follows, and the other parts are similar.

![image.png](../.gitbook/assets/89%20%281%29.png)

It is not difficult to see that RESTStorageProvider is the core component that undertakes configuration to API Group. Such a design can clearly divide the boundaries of each structure and interface, and set a reasonable process.

## Cluster Authentication

### Controller Runner

![master-server-controller-runner.svg](../.gitbook/assets/90%20%281%29.png)

The Listener has only one Enqueue method and is registered somewhere through Notifier. ControllerRunner controls the execution of a task. If it is necessary to notify the outside during the execution process, it will broadcast \(or unicast\) to the target task queue through the registered Listener list. The queue owner may be a task waiting for the queue to output.

Through this design, use the queue feature to isolate the two related tasks and divide their boundaries. The Enqueue method of the Listener interface has no parameters. Therefore, the implementation of the Listener focuses more on the occurrence of the event rather than the specific details of the event content. This idea is worth learning.

### Dynamic CA

![master-server-dynamic-file-ca-content.svg](../.gitbook/assets/91%20%281%29.png)

* PKI certificate and requirements

