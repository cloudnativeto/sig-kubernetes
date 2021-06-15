---
draft: false
profile: 华北电力大学大四学生。
keywords:
  - Kubernetes
  - APIServer
author: '[杨鼎睿](https://yuque.com/abser)'
date: '2021-05-06T17:04:05.000Z'
title: Kubernetes APIServer Cacher 架构设计源码阅读
tags:
  - Kubernetes
  - 源码架构图
  - APIServer
avatar: /images/profile/abserari.png
type: post
image: /images/blog/kubernetes-apiserver.png
categories:
  - kubernetes
description: '本文研究了 Cacher 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。'
---

# API Server Cacher

This paper studies the source code of the Storage section. You should read the source code at the same time. It can enhance your design capacity. 

## Overview

![cacher-overview.svg](../.gitbook/assets/21.png)

Cacher contains an instance of storage.Interface, which is a real storage backend instance. At the same time, Cacher also implements storage.Interface interface, which is a typical **decorator pattern**. There are a large number of elegant **design patterns** in the Kubernetes source code, so you can pay more attention when reading. After simply tracking the code, the current guessed relationship is as follows.

![cacher-storage-interface.svg](../.gitbook/assets/22.png)

The registry package location is as follows.

![image.png](../.gitbook/assets/23.png)

The storage package location is as follows.

![image.png](../.gitbook/assets/24.png)

‌Store initialization code set DryRunnableStorage location.

![image.png](../.gitbook/assets/25.png)

## Store

### Interface Definition

The Store interface is defined in k8s.io/client-go. Pay attention to the Add/Update/Delete in the interface, which is used to add objects to the Store. Then the role of this interface is the glue between API Server and Etcd.

![image.png](../.gitbook/assets/26.png)

![image.png](../.gitbook/assets/27.png)

## Event Main Cycle

![cacher-event-main-cycle.svg](../.gitbook/assets/28.png)

The Cacher structure is defined as follows, which contains a watchCache instance.

![image.png](../.gitbook/assets/29%20%282%29.png)

Look at the Cacher initialization method again. Line 373 is used to create a watchCache instance. The EventHandler passed in is a method of Cacher. In this way, watchCache has a channel for injecting events into Cacher.

![image.png](../.gitbook/assets/30.png)

The dispatchEvents method in the above code seems to be the part that processes the Event sent from the watchCache method. Let's continue, it seems that we are about to solve the event source problem.

![image.png](../.gitbook/assets/31.png)

Keep track of incoming, so does processEvent seem familiar?

![image.png](../.gitbook/assets/32.png)

Go to the watchCache structure and find the place where eventHandler is used.

![image.png](../.gitbook/assets/33.png)

Continue to dig, so far, we have found the complete source of the event, and **there are only three types of events**: Add/Update/Delete.

![image.png](../.gitbook/assets/34.png)

### watchCache.processEvent

The generation of the original event to the final event is shown in the figure below. The keyFunc, getAttrsFunc, Indexer, etc. used are all passed in through configuration.

![cacher-event-generation.svg](../.gitbook/assets/35.png)

After the event created, refresh the cache.

![image.png](../.gitbook/assets/36.png)

## Event Generation

### Cache Watcher

The related structure of cacheWatcher in Cacher is shown in the figure below.

![cacher-cache-watcher.svg](../.gitbook/assets/37.png)

The cacheWatcher implements the watch.Interface interface for monitoring events. The watch.Interface declaration is as follows.

![image.png](../.gitbook/assets/38.png)

The definition of watch.Event is as follows.

![image.png](../.gitbook/assets/39.png)

The core processing flow of cacheWatcher is as follows.

![cacher-cacher-watcher-core.svg](../.gitbook/assets/40.png)

### Watch

#### Cacher

![](../.gitbook/assets/image%20%285%29.png)

The judgment processes of triggerValue and triggerSupported are as follows.

![image.png](../.gitbook/assets/42.png)

CacheWatcher's Input Channel cache size calculation is as follows.

![image.png](../.gitbook/assets/43.png)

The specific addition code is as follows

![image.png](../.gitbook/assets/44.png)

The forgetWatcher is as follows. clean watcher from Cacher.

![image.png](../.gitbook/assets/45.png)

## Event Dispatching

### Bookmark Event

![cacher-bookmark-event.svg](../.gitbook/assets/46.png)

In the Cacher event distribution process, a Timer is created. Each time this Timer is triggered, it is possible to generate a Bookmark Event event and distribute this event. The source code is as follows.

![image.png](../.gitbook/assets/47.png)

#### Dispatch

After the Bookmark Event is created, the ResourceVersion information of the event object is updated through Versioner, and then the event is distributed. Next, let's take a look at how to distribute.

![image.png](../.gitbook/assets/48.png)

The Bookmark Event distribution process is shown in the following figure. You can see that the event has been distributed to all cacheWatchers whose IDs are less than the current time.

![cacher-bookmark-event-dispatch.svg](../.gitbook/assets/49.png)

After arriving at CacheWatcher, the processing is very simple, just returns the original object.

![image.png](../.gitbook/assets/50.png)

### General Event

#### Dispatch

![cacher-general-event-dispatch.svg](../.gitbook/assets/51.png)

As you can see from the figure above, when the length of watchersBuffer is greater than or equal to 3, the object is cached for sending. When sending an event, if there is a failure, get an available time slice, within this time slice, try to block sending the event. If all the transmissions are successful, the waiting time slice is exhausted.

![image.png](../.gitbook/assets/52.png)

If sending fails within the time slice, delete the remaining cacheWatcher.

![image.png](../.gitbook/assets/53.png)

## References

* [https://kubernetes.io/docs/reference/using-api/api-concepts/](https://kubernetes.io/docs/reference/using-api/api-concepts/)

