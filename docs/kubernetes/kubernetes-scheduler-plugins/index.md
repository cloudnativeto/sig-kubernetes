---
title: "Kubernetes Scheduler Plugins 架构设计源码阅读"
date: 2021-04-27T02:04:05+08:00
draft: false
image: "/images/blog/kubernetes-scheduler.png"
author: "[杨鼎睿](https://yuque.com/abser)"
description: "本文研究了 Kubernetes 中 Plugins 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。"
tags: ["Kubernetes","源码架构图", "Scheduler"]
categories: ["kubernetes"]
keywords: ["Kubernetes","Scheduler"]
type: "post"
avatar: "/images/profile/abserari.png"
profile: "华北电力大学大四学生。"
---
本文研究了 Kubernetes 中 Plugins 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。
<a name="Jm4Jk"></a>
## Basic Structures
<a name="Plugin"></a>
### Plugins
![kube-scheduler-plugin-plugins.svg](1.png)

Plugins 包含了一组 PluginSet，每个 PluginSet 又包含了两组 Plugin，一组为激活状态，一组为禁用状态。Plugin 包含一个唯一标识符和该 Plugin 的权重。

<a name="nIs6M"></a>
### Quick Sort Plugin
![kube-scheduler-plugin-instances-quicksort-less.svg](2.png)

接口方法 Less 如上图所示，传入两个 PodInfo 实例，通过 Less 方法判定二者在排序时先后位置。由于 Plugin 接口只有通用方法 Name，因此，每个特定功能的 Plugin 原则上可随意定制自己需要的方法。

<a name="FCpuJ"></a>
### InterPodAffinity
![kube-scheduler-plugin-instances-inter-pod-affinity-pre-filter.svg](3.png)

将与当前调度 Pod 的相关 Node 数据存储在 CycleState 的 PreFilterInterPodAffinity 关键字中，供后续调度使用。

<a name="U4bQo"></a>
## Configurator
<a name="U2mlC"></a>
### Profile
Registry 用于组织 provider 与 Plugins 间关联关系，保存的是默认的 Plugins。在创建 Profile 结构时，会使用这些默认的 Plugins。

![kube-scheduler-plugin-configurator-profile.svg](4.png)



<a name="8QFVr"></a>
#### 


<a name="n2OjD"></a>
### Extender
![kube-scheduler-plugin-extender.svg](5.png)




相关代码如下图所示

![image.png](6.png)



<a name="17Xnt"></a>
### Profile Map
![kube-scheduler-plugin-framework-relation.svg](7.png)


本文研究了 Kubernetes 中 Plugins 部分的源码，是 Scheduler 的第三部分。