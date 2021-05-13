---
title: "Kubernetes Scheduler Priority Queue 架构设计源码阅读"
date: 2021-04-27T02:04:05+08:00
draft: false
image: "/images/blog/kubernetes-scheduler.png"
author: "[杨鼎睿](https://yuque.com/abser)"
description: "本文研究了 Kubernetes 中 Priority Queue 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。"
tags: ["Kubernetes","源码架构图", "Scheduler"]
categories: ["kubernetes"]
keywords: ["Kubernetes","Scheduler"]
type: "post"
avatar: "/images/profile/abserari.png"
profile: "华北电力大学大四学生。"
---
本文研究了 Kubernetes 中 Priority Queue 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。
<a name="tKXPW"></a>
## Internal Data Structures
<a name="Heap"></a>
### Heap
![kube-scheduler-priority-queue-heap.svg](1.png)

上图为 PriorityQueue 中 activeQ 域，它是一个 Heap 实例。Heap 中核心结构为 data，包含一个字符串到 heapItem 的映射，heapItem 存储了实际对象以及该对象的 Key 在 queue 中位置的变量。

<a name="20006f39"></a>
### Pods Containers
![kube-scheduler-pods-map.svg](2.png)

PriorityQueue 中与 Pod 关联的两个数据结构如上图所示。UnschedulablePodsMap 中保存了从 Pod 信息到 key 值的方法。

<a name="nominatedPodMap"></a>
### nominatedPodMap
<a name="Add"></a>
#### Add
![kube-scheduler-nominated-pod-map-add.svg](3.png)

优先使用传入的 nodeName，若 nodeName 为空时，使用 UID，若 UID 也为空，处理完毕。否则，按上图示意，添加对应的 map。

<a name="m3ENt"></a>
## Pod Handling
<a name="42jZk"></a>
### Added/Update
![kube-scheduler-priority-queue-get-unschedulable-pods.svg](4.png)

kube-scheduler 接收到添加 Pod 事件后，会将 Pod 添加进 SchedulerCache 中，随后，执行上图操作。Pod 的添加、更新操作执行相同代码，区别为状态不同，分别对应为 AssignedPodAdd 与 AssignedPodUpdate。

SchedulerCache 注
```go
// State Machine of a pod's events in scheduler's cache:
//
//
//   +-------------------------------------------+  +----+
//   |                            Add            |  |    |
//   |                                           |  |    | Update
//   +      Assume                Add            v  v    |
//Initial +--------> Assumed +------------+---> Added <--+
//   ^                +   +               |       +
//   |                |   |               |       |
//   |                |   |           Add |       | Remove
//   |                |   |               |       |
//   |                |   |               +       |
//   +----------------+   +-----------> Expired   +----> Deleted
//         Forget             Expire
```

<a name="Zn3vg"></a>
### Delete
![kube-scheduler-priority-queue-move-all-to-active-backoff.svg](5.png)

将 podInfoMap 中全部 PodInfo 移动至 podBackoffQ 或 activeQ 中，并删除 podInfoMap 中对应 K/V 对，标记状态为 AssignedPodDelete。

<a name="k74pB"></a>
## Handling Routines
<a name="heeeN"></a>
### Flush Backoff Queue
![kube-scheduler-priority-queue-flush-backing-off.svg](6.png)

定时从 backoff queue 中获取一个 PodInfo 对象，检查其 backoff time 是否到期，如果没有到期，直接返回，等待下次触发，此时，PodInfo 对象仍然存在于 backoff queue 中。如果到期，则弹出该对象，并存入 active queue 中，此时，PodInfo 对象被移除出 backoff queue。

<a name="ffttC"></a>
### Flush Unscheduable Queue
![kube-scheduler-priority-queue-flush-unschedulable.svg](7.png)

本文研究了 Kubernetes 中 Priority Queue 部分的源码，是 Scheduler 的第二部分。