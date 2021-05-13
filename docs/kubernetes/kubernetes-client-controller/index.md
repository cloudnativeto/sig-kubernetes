---
title: "Kubernetes Client Controller 架构设计源码阅读"
date: 2021-04-27T02:04:05+08:00
draft: false
image: "/images/blog/kubernetes-client.png"
author: "[杨鼎睿](https://yuque.com/abser)"
description: "本文研究了 Kubernetes 中 Client 中 Controller 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。"
tags: ["Kubernetes","源码架构图", "Client"]
categories: ["kubernetes"]
keywords: ["Kubernetes","Client"]
type: "post"
avatar: "/images/profile/abserari.png"
profile: "华北电力大学大四学生。"
---
本文研究了 Kubernetes 中 Client 中 Controller  部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。
<a name="3pa4L"></a>
## Controller
Controller 内部工作过程为：注册 ResourceEventHandler，事件到达后，存入 Controller 内部 Queue。Worker 协程周期性的从 Queue 中弹出对象，并处理。

![controller-controller.svg](1.png)



<a name="ONkHS"></a>
### Event Dispatch
Controller 注册的 ResourceEventHandler 将事件分发。分发过程如下图，EventBroadcaster 创建 EventRecorder，事件在 ResourceEventHandler 的处理过程中，根据各自处理逻辑决定是否分发至 EventRecorder 中；如果分发至 EventRecorder，则通过 EventBroadcaster 分发至 watch.Interface。




watch.Interface 根据需要注册在 EventBroadcaster 上，通过 EventBroadcaster 实现对事件的 watch 功能。EventBroadcaster 分发事件时，可选择阻塞式发送方式；也可选择非阻塞式发送，如果发送失败，则 watch.Interface 实例会丢失该事件。

![controller-event-watch.svg](2.png)

EventBroadcaster 启动 Sink，将 watch.Interface 中获取的事件，回传至 API Server，并记录为 Event 事件。

![image.png](3.png)




从上面的分析看：

- EventResourceHandler 是核心处理过程
- EventResourceHandler 在处理过程中，需要其他 Controller 配合、动作事件回传至 API Server
- EventBroadcaster 及 Sink 的作用是将 EventResourceHandler 生成的其他数据，回传至 API Server



<a name="bLxD3"></a>
## ResourceEventHandler
ResourceEventHandler 处理资源变更事件，该接口包含了 OnAdd/OnUpdate/OnDelete 三个方法。

<a name="ET8PS"></a>
### Filter Enhancement
![controller-resource-event-handler.svg](4.png)

- FilteringResourceEventHandler 实现了 ResourceEventHandler 接口
- 包含一个 FilterFunc 和一个 ResourceEventHandler 实例
- OnAdd/OnDelete 时
   - 先执行 FilterFunc，如果失败则退出
   - 触发 ResourceEventHandler
- OnUpdate 时
   - 分别对传入的 old、new 对象执行 FilterFunc
   - old，new 均通过 FilterFunc，触发 ResourceEventHandler 的 OnUpdate  方法
   - old 被过滤，触发 ResourceEventHandler 的 OnAdd  方法
   - new 被过滤，触发 ResourceEventHandler 的 OnDelete  方法



本文研究了 Kubernetes 中 Client 中 Controller  部分的源码，是 Client 的第二部分。