---
draft: false
profile: 华北电力大学大四学生。
keywords:
  - Kubernetes
  - Controller
author: '[杨鼎睿](https://yuque.com/abser)'
date: '2021-05-10T05:04:05.000Z'
title: Kubernetes Controller Controllers 架构设计源码阅读
tags:
  - Kubernetes
  - 源码架构图
  - Controller
avatar: /images/profile/abserari.png
type: post
image: /images/blog/kubernetes-controller.png
categories:
  - kubernetes
description: '本文研究了 Controllers 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。'
---

# Controllers

大家好，我是杨鼎睿，这一次给大家带来的是 Controllers 的源码阅读。

## Starting Controller

Controller 启动过程是类似的，首先创建到 API Server 的客户端连接 clientset.Interface，它包含了访问 API Server 不同类型资源的客户端。

然后，启动 SharedInformer 接口实例，伴随其启动的，还有一个 Controller 实例。Controller 定期从 API Server 获取资源变更，并存入 Store 实例中。Controller 的 processLoop 协程，从 Store 中顺序读取资源变更事件，并交由 sharedIndexInformer 实例处理，最终到达 ResourceEventHandler。

Controller 实现的核心，在于其对监听资源的变更处理方法上。

![controller-mode.svg](../.gitbook/assets/5%20%283%29.png)

简化后的工作流如下图

![controller-workflow.svg](../.gitbook/assets/6%20%283%29.png)

## Controller Manager

Controller Manager 负责启动 Controllers。通过注册不同类型 Controller 的初始化方法，并创建 ControllerContext，隔离了 Controller 具体实现。

**Controller Manager ---Create--&gt; ControllerContext ---Pass--&gt; Initialization Function**

![controller-controller-manager.svg](../.gitbook/assets/7%20%283%29.png)

1.18 版本，注册的 Controller

![image.png](../.gitbook/assets/8%20%283%29.png)

\[6\] Controllers

* [Controllers Queue](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-controller-queue/README.md)
* [Controllers](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-controller-controllers/README.md)
* [Endporint Controller](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-controller-endpoint-controller/README.md)
* [Namespace Controller](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-controller-namespace-controller/README.md)
* [Node Controllers](https://github.com/cloudnativeto/sig-kubernetes/tree/f0b2470abda40d4c0ac2b727df5562b4f2cf996e/blog/kubernetes-controller-node-controllers/README.md)

