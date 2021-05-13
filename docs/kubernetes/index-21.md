# Proxy Framework

## References

* [Feature Gates](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
* [Service Networking](https://kubernetes.io/docs/concepts/services-networking/service/)

## Setup

### Events

![kube-proxy-broadcaster.svg](../.gitbook/assets/1%20%288%29.png)

* 创建 Broadcaster
* 使用 Broadcaster 创建 Recorder
* 各组件根据自己需要，使用 Recorder 向 Broadcaster 发送 Event

### Informers

![kube-proxy-informers.svg](../.gitbook/assets/2%20%289%29.png)

  
Service Informer 是一定存在的 Informer；EndpointSlice 及 Endpoint 二选一，1.18 默认使用 Ednpoit Informer；Node Informer 需要开启 ServiceTopology 选项。最终，都由 Provider 来处理各类事件。  
具体来说，每一类型资源都会创建独自的 ResourceEventHandler。各自的 Handler 都是一个 slice 类型，slice 内存储真正的资源 Handler，如下图所示  
 

![kube-proxy-resource-handler.svg](../.gitbook/assets/3%20%289%29.png)



### Bounded Frequency Runner

![kube-proxy-bounded-frequency-runner.svg](../.gitbook/assets/4%20%289%29.png)

