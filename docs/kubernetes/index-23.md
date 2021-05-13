# Proxy IPVS

## IPVS

### References

* [IPVS-Based In-Cluster Load Balancing Deep Dive](https://kubernetes.io/blog/2018/07/09/ipvs-based-in-cluster-load-balancing-deep-dive/)
* [IP-SYSCTL](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html)
* [IPVS-SYSCTL](https://www.kernel.org/doc/html/latest/networking/ipvs-sysctl.html)
* [Martian Packet](https://en.wikipedia.org/wiki/Martian_packet)
* [Linux Kernel Networking](https://www.kernel.org/doc/html/latest/networking/index.html)

### System Control Configuration

* /proc/sys/net/ipv4/conf/all/route\_localnet ---&gt; 1

![image.png](../.gitbook/assets/27%20%282%29.png)

* /proc/sys/net/bridge/bridge-nf-call-iptables ---&gt; 1

![image.png](../.gitbook/assets/28%20%282%29.png)

* /proc/sys/net/ipv4/vs/conntrack ---&gt; 1

![image.png](../.gitbook/assets/29%20%284%29.png)

* /proc/sys/net/ipv4/vs/conn\_reuse\_mode ---&gt; 0

![image.png](../.gitbook/assets/30%20%283%29.png)

* /proc/sys/net/ipv4/vs/expire\_nodest\_conn ---&gt; 1

![image.png](../.gitbook/assets/31%20%283%29.png)

* /proc/sys/net/ipv4/vs/expire\_quiescent\_template ---&gt; 1

![image.png](../.gitbook/assets/32%20%283%29.png)

* /proc/sys/net/ipv4/ip\_forward ---&gt; 1

![image.png](../.gitbook/assets/33%20%283%29.png)

* StrictARP
  * /proc/sys/net/ipv4/conf/all/arp\_ignore ---&gt; 1

![image.png](../.gitbook/assets/34%20%282%29.png)

* /proc/sys/net/ipv4/conf/all/arp\_announce ---&gt; 2

![image.png](../.gitbook/assets/35%20%282%29.png)

## Sync Services

### Basic Chains & Jump

![ipvs-proxy-basic-chains.svg](../.gitbook/assets/36%20%282%29.png)

### Cluster IP Handling

![ipvs-proxy-ipvs-entry.svg](../.gitbook/assets/37%20%282%29.png)

#### Create Virtual Server

使用 Service 的 ClusterIP、Port 等信息，创建 VirtualServer，通过 netlink 来查询、创建、更新、删除 VirtualServer。  
 

![ipvs-proxy-ipvs-create-update-server.svg](../.gitbook/assets/38%20%282%29.png)

  


### External IP Handling

![ipvs-proxy-external-ips.svg](../.gitbook/assets/39%20%282%29.png)

  
VirtualServer 通过 netlink 接口创建 libipvs.Service 与处理 ClusterIP 类型服务时相同。需要注意，如果开启特性 ExternalPolicyForExternalIP，并且当前处理的 Service 的 Endpoint 只存在与当前 Node，那么使用 KUBE-EXTERNAL-IP-LOCAL 存储 Entry 值。

#### Query Destinations

![ipvs-proxy-find-real-server.svg](../.gitbook/assets/40%20%283%29.png)

  
在同步 Endpoints 前，需要获取当前服务的全部 RealServer，可通过 netlink 根据 VirtualServer 获取其对应的 Destination 列表，再根据 Destination 转化为 RealServer

```go
func toRealServer(dst *libipvs.Destination) (*RealServer, error) {
    if dst == nil {
        return nil, errors.New("ipvs destination should not be empty")
    }
    return &RealServer{
        Address:      dst.Address,
        Port:         dst.Port,
        Weight:       dst.Weight,
        ActiveConn:   dst.ActiveConnections,
        InactiveConn: dst.InactiveConnections,
    }, nil
}
```

#### Synchronize Endpoints

![ipvs-proxy-sync-endpoints.svg](../.gitbook/assets/41%20%281%29.png)

  
处理过程比较简单，先处理 newEndpoints 中新增或更新的 Endpoint。处理完毕后，将处理删除的 Endpoints，被移除的 IP 及 Port 可通过 curEndpoints.Difference\(newEndpoints\) 获取。遍历删除列表，如果 IP、Port 已存在于 termination list，则不需要任何处理；如果不存在，将该 IP、Port 存入 termination list，同时存入的还有其对应的 netlink 创建的 Server。

### Load Balancer Handling

![ipvs-proxy-load-balancer.svg](../.gitbook/assets/42%20%281%29.png)

### Node Port

![ipvs-proxy-node-port.svg](../.gitbook/assets/43%20%281%29.png)

  
找到本地地址，并根据当前 Service 使用的 NodePort 情况，创建监听的端口。遍历时，如果遇到零地址段，则退出循环，因为这意味着本机全部 IP 都要监听该 NodePort。  
创建 Entry 结构时，与当前 Service 的协议相关，如果为 SCTP 协议，类型为：HashIPPort。如果当前 Service 不是 Node Only，则只需要添加至 KUBE-NODE-PORT-_**protocol**_ 中即可。

### Synchronize IPSet

![ipvs-proxy-sync-ipset.svg](../.gitbook/assets/44%20%281%29.png)

  
在之前的处理中，在 ipsetList 中各种类型的 IPSet 上，添加了各自的 IP、端口信息，在这个方法中将其应用。utilipset.Interface 实现是基于 ipset 命令集的，ListEntries 方法中传入的 Name 是内部的 utilipset.IPSet 中 Name 域，注意区分。

### IPTables Rules

在之前的处理中 RealServer 已创建完毕，但是，每个 Node 只创建了一个 dummy 类型的网络设备 kube-ipvs0，那么，需要通过 iptables 及 ipset 配置，将流量合理引导。  
首先创建的是如下的 NAT 规则，因为使用 -A 选择，下图中顺序即规则顺序，现在需要添加规则，从系统内置链中跳转至不同 Chain 中处理。  
 

![ipvs-proxy-iptables-with-ipset.svg](../.gitbook/assets/45%20%281%29.png)

  
跳转规则如下所示  
 

![ipvs-proxy-kube-chains.svg](../.gitbook/assets/46%20%281%29.png)



