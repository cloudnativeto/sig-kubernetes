---
draft: false
profile: 华北电力大学大四学生。
keywords:
  - Kubernetes
  - APIServer
author: '[杨鼎睿](https://yuque.com/abser)'
date: '2021-05-01T18:04:05.000Z'
title: Kubernetes APIServer ETCD 架构设计源码阅读
tags:
  - Kubernetes
  - 源码架构图
  - APIServer
avatar: /images/profile/abserari.png
type: post
image: /images/blog/kubernetes-apiserver.png
categories:
  - kubernetes
description: >-
  This article has studied the source code of the ETCD part, equipped with the
  source code for further understanding, which can deepen the understanding and
  enhance related design capabilities.
---

# API Server Etcd

Hello everyone, this time I bring you ETCD source code reading. The three parts of this article are the Server part, the Storage part, and the Utility part. With the source code for further understanding, you can deepen your understanding and enhance related design capabilities.

## Server

### Single

#### Landscape

![landscape.svg](../.gitbook/assets/1%20%281%29.png)

Clients contain the address to be monitored by the etcd server. The address can be in the form of TCP or Unix Socket and supports http and https. The [serverCtx](https://sourcegraph.com/github.com/etcd-io/etcd@release-3.3/-/blob/embed/serve.go#L46:6) matches a net.Listener and runs independently of a goroutine.

#### Serve Procedure

![serve-clients.svg](../.gitbook/assets/2%20%281%29.png)

### Backend

#### Landscape

![backend-landscape.svg](../.gitbook/assets/3%20%282%29.png)

## Storage

### BoltDB Backend

#### Landscape

![bolt-backend.svg](../.gitbook/assets/4%20%282%29.png)

**run**

Start Timer regularly submits or submit and exit when receiving the stop signal. The code is simple, as shown below.

```go
func (b *backend) run() {
    defer close(b.donec)
    t := time.NewTimer(b.batchInterval)
    defer t.Stop()
    for {
        select {
        case <-t.C:
        case <-b.stopc:
            b.batchTx.CommitAndStop()
            return
        }
        b.batchTx.Commit()
        t.Reset(b.batchInterval)
    }
}
```

**Transaction Relationship**

![tx-relation.svg](../.gitbook/assets/5%20%282%29.png)

**Buffer**

![tx-buffer.svg](../.gitbook/assets/6%20%282%29.png)

### MVCC

#### Store

![store.svg](../.gitbook/assets/7%20%282%29.png)

**References**

* [ReadView](https://sourcegraph.com/github.com/etcd-io/etcd@release-3.3/-/blob/mvcc/kv.go#L35:6)
* [WriteView](https://sourcegraph.com/github.com/etcd-io/etcd@release-3.3/-/blob/mvcc/kv.go#L63:6)
* [KV](https://sourcegraph.com/github.com/etcd-io/etcd@release-3.3/-/blob/mvcc/kv.go#L100:6)

#### Watchable

**Landscape**

![watchable.svg](../.gitbook/assets/8%20%282%29.png)

**Watcher Creation**

![watcher-creation.svg](../.gitbook/assets/9%20%282%29.png)

The ch of all watchers created through watchStream all point to watchStream.ch. The event direction is watcher -&gt; watchStream. Watcher management is in charge of watcherGroup.![watcher-group.svg](../.gitbook/assets/10%20%281%29.png)

**Nofity Waiter**

![notify-waiter.svg](../.gitbook/assets/11%20%281%29.png)

## Utility

### CMux

* [soheilhy/cmux](https://github.com/soheilhy/cmux): Connection multiplexer for GoLang: serve different services on the same port!

#### Landscape

![landscape.svg](../.gitbook/assets/12.png)

#### Implementation

![cmux.svg](../.gitbook/assets/13.png)

### Scheduler

#### Landscape

![utility.svg](../.gitbook/assets/14.png)

### WAL

#### Landscape

![wal.svg](../.gitbook/assets/15.png)

