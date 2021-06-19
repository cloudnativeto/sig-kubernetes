---
draft: false
profile: 华北电力大学大四学生。
keywords:
  - Kubernetes
  - APIServer
author: '[杨鼎睿](https://yuque.com/abser)'
date: '2021-05-06T17:04:05.000Z'
title: Kubernetes APIServer CRD 架构设计源码阅读
tags:
  - Kubernetes
  - 源码架构图
  - APIServer
avatar: /images/profile/abserari.png
type: post
image: /images/blog/kubernetes-apiserver.png
categories:
  - kubernetes
description: '本文研究了 CRD 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。'
---

# API Server CustomResourceDefinitions

This article has studied the source code of the CRD part, equipped with the source code for further understanding, which can deepen the understanding and enhance related design capabilities. 

## ResourceConfig

### Default Configuration

![image.png](../.gitbook/assets/60.png)

Enabled resource configuration and disabled version.

![image.png](../.gitbook/assets/61.png)

### Extend

![image.png](../.gitbook/assets/62.png)

Enabled the selection as follows.

![image.png](../.gitbook/assets/63.png)

## Runtime Support

![image.png](../.gitbook/assets/64.png)

The three are shown below.

![api-extension-server-runtime-support.svg](../.gitbook/assets/65.png)

### Storage

#### Custom Resource Definitions

![api-extension-server-rest-storage.svg](../.gitbook/assets/66.png)

The Store is expanded as shown below.

![api-extension-server-rest-store.svg](../.gitbook/assets/67.png)

## State Transition

### Landscape

SharedInformerFactory is used to create SharedIndexInformer, which will periodically use Clientset to connect to the API Extension Services of v1beta1 or v1 and notify the respective ResourceEventHandler after obtaining the status change. Here, there are still some issues that need to dig deeper:

* How SharedInformerFactory distinguishes different types of resource state changes
* Can ResourceEventHandler pay attention to changes in the state of different types of resources at the same time
* How are resource status changes obtained

![api-extension-server-shared-informer-relations.svg](../.gitbook/assets/68.png)

### Clientset

The Clientset function is relatively simple. It encapsulates the available API Extension Services. Each RESTClient is connected to the "Loopback" address and sends requests to different services.

![api-extension-server-clientset.svg](../.gitbook/assets/69.png)

### SharedInformerFactory

#### Relationship

![api-extension-server-shared-informer-workflow.svg](../.gitbook/assets/70.png)

#### Add Informers

![api-extension-server-add-informer.svg](../.gitbook/assets/71.png)

## Management

### EstablishingController

After the EstablishingController is started, it will start a scheduled execution task. This task checks every second whether there is a new **Key** value in the queue. If there is, update the corresponding resource status on the Server side to **Established**.

![api-extension-server-establishing-controller.svg](../.gitbook/assets/72.png)

The sync code is as follows.

![image.png](../.gitbook/assets/73.png)

### CRD Handler

![api-extension-server-crd-controller.svg](../.gitbook/assets/74.png)

The CRD Handler registers event processing with SharedIndexInformer. When the Watch object type is Update, it may be that the state changes to the Established state and needs to be sent to the EstablingController.

When the CRD Handler processes the request, it first checks whether the cache contains the requested object, if so, returns the cached object; if not, it requests the Server and changes the cache status. 

### CRD Controller

![api-extension-server-crd-controller.svg](../.gitbook/assets/75.png)

