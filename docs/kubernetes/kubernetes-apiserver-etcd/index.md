---
title: "Kubernetes APIServer ETCD 架构设计源码阅读"
date: 2021-05-02T02:04:05+08:00
draft: false
image: "/images/blog/kubernetes-apiserver.png"
author: "[杨鼎睿](https://yuque.com/abser)"
description: "本文研究了 ETCD 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。"
tags: ["Kubernetes","源码架构图", "APIServer"]
categories: ["kubernetes"]
keywords: ["Kubernetes","APIServer"]
type: "post"
avatar: "/images/profile/abserari.png"
profile: "华北电力大学大四学生。"
---

大家好，我是杨鼎睿，这一次给大家带来的是 ETCD 的源码阅读。本文写就时是三部分，方便大家阅读，合成一篇，分别是 Server 篇， Storage 篇和 Utility 篇。

本文研究了 ETCD 部分的源码，配备源码进行进一步理解，可以加深理解,增强相关设计能力。

# Server
<a name="oDDrO"></a>
## Single
<a name="n9ox4"></a>
### Landscape
![landscape.svg](1.png)

Clients 中包含了 etcd 服务器要监听的地址，地址可为 TCP、Unix Socket 形式，且支持 http 与 https。[serverCtx](https://sourcegraph.com/github.com/etcd-io/etcd@release-3.3/-/blob/embed/serve.go#L46:6) 与一个 net.Listener 匹配，独立运行于一个 goroutine。


<a name="Oslrv"></a>
### Serve Procedure
![serve-clients.svg](2.png)

<a name="TCL9H"></a>
## Backend
<a name="xqINa"></a>
### Landscape
![backend-landscape.svg](3.png)

# Storage
<a name="RwQHd"></a>
## BoltDB Backend
<a name="oj1bO"></a>
### Landscape
![bolt-backend.svg](4.png)

<a name="wptN1"></a>
#### run
启动 Timer，定时提交；或者在收到停止信号时，提交并退出。代码很简单，如下所示
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

<a name="kmnwM"></a>
#### Transaction Relationship
![tx-relation.svg](5.png)

<a name="3UU3i"></a>
#### Buffer
![tx-buffer.svg](6.png)

<a name="QVPPC"></a>
## MVCC
<a name="wMnDW"></a>
### Store
![store.svg](7.png)

<a name="4vc0c"></a>
#### References

- [ReadView](https://sourcegraph.com/github.com/etcd-io/etcd@release-3.3/-/blob/mvcc/kv.go#L35:6)
- [WriteView](https://sourcegraph.com/github.com/etcd-io/etcd@release-3.3/-/blob/mvcc/kv.go#L63:6)
- [KV](https://sourcegraph.com/github.com/etcd-io/etcd@release-3.3/-/blob/mvcc/kv.go#L100:6)

<a name="KGHho"></a>
### Watchable
<a name="Jti9a"></a>
#### Landscape
![watchable.svg](8.png)

<a name="HxUXg"></a>
#### Watcher Creation
![watcher-creation.svg](9.png)

通过 watchStream 创建的全部 watcher 的 ch 全部指向了 watchStream.ch。事件走向为 watcher -> watchStream。watcher 管理由 watcherGroup 负责
![watcher-group.svg](10.png)


<a name="19sN5"></a>
#### Nofity Waiter
![notify-waiter.svg](11.png)

# Utility
<a name="zjfFl"></a>
## CMux
- [soheilhy/cmux](https://github.com/soheilhy/cmux): Connection multiplexer for GoLang: serve different services on the same port!

<a name="rYHMJ"></a>
### Landscape
![landscape.svg](12.png)

<a name="VA3hj"></a>
### Implementation
![cmux.svg](13.png)

<a name="sQtm8"></a>
## Scheduler
<a name="NNRs9"></a>
### Landscape
![utility.svg](14.png)

<a name="OSuKs"></a>
## WAL
<a name="RjwDB"></a>
### Landscape
![wal.svg](15.png)

致谢一下响哥（李响）在我阅读 bboltDB 的源码的时候给予我的鼓励，使我对 ETCD 的源码产生兴趣（可惜到现在还没有见过一面）。本篇实际是在阅读 API Server 的时候完成的，本意也是为了更好的理解 API Server，API Server 部分图较多，也经历过多次重画，最近整理完毕会逐步放出，有意阅读交流的朋友请持续关注，或者催更（笑）。另外我时常会组织一起进行源码阅读并画图的活动（通常是我和我对象，本篇的 Server 部分就是她画的），如果大家有兴趣的话也可以加入进来对一些优秀设计的开源项目进行源码阅读和画图分享，表示欢迎。

[3] API Server
- [API Server Routes](/blog/kubernetes-apiserver-route/)
- [API Server API Group](/blog/kubernetes-apiserver-apigroup/)
- [API Server Storage](/blog/kubernetes-apiserver-storage/)
- [API Server Cacher](/blog/kubernetes-apiserver-cacher/)
- [API Server Etcd](/blog/kubernetes-apiserver-etcd/)
- [API Server Generic API Server](/blog/kubernetes-apiserver-generic-api-server/)
- [API Server CustomResourceDefinitions](/blog/kubernetes-apiserver-crd/)
- [API Server Master Server](/blog/kubernetes-apiserver-master-server/)
- [API Server Aggregator Server](/blog/kubernetes-apiserver-aggregator-server/)
- [API Server API Server Deprecated (暂无)](/blog/kubernetes-apiserver-route/)
