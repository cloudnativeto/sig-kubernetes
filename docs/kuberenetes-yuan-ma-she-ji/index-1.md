# Docker Networking

## Design

![image.png](../.gitbook/assets/1%20%287%29.jpeg)

* Sandbox: 协议栈，可包含多个 Endpoint，可通过 Namespace、Jail 等实现
* Endpoint: 将 Sandbox 与 Network 连接
* Network: 可直接通信的 Endpoint 的集合，可使用 Bridge、VLAN 等实现

## Docker Architecture

![docker-network-arch.svg](../.gitbook/assets/2%20%284%29.jpeg)

## Network Controller

![docker-network-network-controller.svg](../.gitbook/assets/3%20%284%29.jpeg)

### Initialize Network Controllers

Docker Daemon 管理可用的 NetworkController。在启动 Daemon 时，会创建当前操作系统下全部可用的 NetworkController，以 daemon\_unix.go 为例，创建了 none、host、bridge 三种模式的网络控制器。

```go
func (daemon *Daemon) initNetworkController(config *config.Config, activeSandboxes map[string]interface{}) (libnetwork.NetworkController, error) {
    netOptions, err := daemon.networkOptions(config, daemon.PluginStore, activeSandboxes)
    if err != nil {
        return nil, err
    }

    controller, err := libnetwork.New(netOptions...)
    if err != nil {
        return nil, fmt.Errorf("error obtaining controller instance: %v", err)
    }

    if len(activeSandboxes) > 0 {
        logrus.Info("There are old running containers, the network config will not take affect")
        return controller, nil
    }

    // Initialize default network on "null"
    if n, _ := controller.NetworkByName("none"); n == nil {
        if _, err := controller.NewNetwork("null", "none", "", libnetwork.NetworkOptionPersist(true)); err != nil {
            return nil, fmt.Errorf("Error creating default \"null\" network: %v", err)
        }
    }

    // Initialize default network on "host"
    if n, _ := controller.NetworkByName("host"); n == nil {
        if _, err := controller.NewNetwork("host", "host", "", libnetwork.NetworkOptionPersist(true)); err != nil {
            return nil, fmt.Errorf("Error creating default \"host\" network: %v", err)
        }
    }

    // Clear stale bridge network
    if n, err := controller.NetworkByName("bridge"); err == nil {
        if err = n.Delete(); err != nil {
            return nil, fmt.Errorf("could not delete the default bridge network: %v", err)
        }
        if len(config.NetworkConfig.DefaultAddressPools.Value()) > 0 && !daemon.configStore.LiveRestoreEnabled {
            removeDefaultBridgeInterface()
        }
    }

    if !config.DisableBridge {
        // Initialize default driver "bridge"
        if err := initBridgeDriver(controller, config); err != nil {
            return nil, err
        }
    } else {
        removeDefaultBridgeInterface()
    }

    // Set HostGatewayIP to the default bridge's IP  if it is empty
    if daemon.configStore.HostGatewayIP == nil && controller != nil {
        if n, err := controller.NetworkByName("bridge"); err == nil {
            v4Info, v6Info := n.Info().IpamInfo()
            var gateway net.IP
            if len(v4Info) > 0 {
                gateway = v4Info[0].Gateway.IP
            } else if len(v6Info) > 0 {
                gateway = v6Info[0].Gateway.IP
            }
            daemon.configStore.HostGatewayIP = gateway
        }
    }
    return controller, nil
}
```

### NetworkController Implementation

![docker-network-network-controller-impl.svg](../.gitbook/assets/4%20%284%29.jpeg)  
controller 是 libnetwork 中对 NetworkController 的实现。可以看到，controller 通过驱动表来区分不同类型的网络，使用驱动创建 Network 及 Endpoint，并将 Endpoint 加入 Sandbox 或移除出 Sandbox。  
Container 通过 SandboxID 以及 SandboxKey 来找到对应的 Sandbox。Sandbox 可以使用 containerID 来确定是否归属于某个 Container。

## OS Layer Sandbox

### Namespace

![docker-network-sandbox.svg](../.gitbook/assets/5%20%283%29.jpeg)  
Sandbox 接口没有列举出全部功能，只是能看出其能力边界的部分功能。后续以 Namespace 方式实现的 Sandbox 为例。  
通过上图，并不难看出，路由、接口等功能应该是由 netlink 提供的，Namespace 获取 netlink 方式如下，需要注意，Namespace 内 netlink 配置，仅在 Namespace 内有效。根据 Namespace 获取 netlink 的关键方法如下

```go
func GetFromPath(path string) (NsHandle, error) {
    fd, err := syscall.Open(path, syscall.O_RDONLY, 0)
    if err != nil {
        return -1, err
    }
    return NsHandle(fd), nil
}
```

使用返回的 NsHandle 就可以创建具体的 SocketHandle 了，方法如下

```go
func NewHandleAt(ns netns.NsHandle, nlFamilies ...int) (*Handle, error) {
    return newHandle(ns, netns.None(), nlFamilies...)
}

// NewHandleAtFrom works as NewHandle but allows client to specify the
// new and the origin netns Handle.
func NewHandleAtFrom(newNs, curNs netns.NsHandle) (*Handle, error) {
    return newHandle(newNs, curNs)
}

func newHandle(newNs, curNs netns.NsHandle, nlFamilies ...int) (*Handle, error) {
    h := &Handle{sockets: map[int]*nl.SocketHandle{}}
    fams := nl.SupportedNlFamilies
    if len(nlFamilies) != 0 {
        fams = nlFamilies
    }
    for _, f := range fams {
        s, err := nl.GetNetlinkSocketAt(newNs, curNs, f)
        if err != nil {
            return nil, err
        }
        h.sockets[f] = &nl.SocketHandle{Socket: s}
    }
    return h, nil
}
```

#### Add Interface

![docker-network-add-interface.svg](../.gitbook/assets/6%20%283%29.jpeg)

## Bridge Network

![docker-network-bridge-overview.svg](../.gitbook/assets/7%20%283%29.jpeg)

### Create Network

根据配置文件中 BridgeName 查找系统中已存在的 Link 实例，如果 BridgeName 为空，使用默认网桥 docker0。

```go
func newInterface(nlh *netlink.Handle, config *networkConfiguration) (*bridgeInterface, error) {
    var err error
    i := &bridgeInterface{nlh: nlh}

    // Initialize the bridge name to the default if unspecified.
    if config.BridgeName == "" {
        config.BridgeName = DefaultBridgeName
    }

    // Attempt to find an existing bridge named with the specified name.
    i.Link, err = nlh.LinkByName(config.BridgeName)
    if err != nil {
        logrus.Debugf("Did not find any interface with name %s: %v", config.BridgeName, err)
    } else if _, ok := i.Link.(*netlink.Bridge); !ok {
        return nil, fmt.Errorf("existing interface %s is not a bridge", i.Link.Attrs().Name)
    }
    return i, nil
}
```

创建 bridgeNetwork 实例，并存入 networks

```go
// Create and set network handler in driver
network := &bridgeNetwork{
    id:         config.ID,
    endpoints:  make(map[string]*bridgeEndpoint),
    config:     config,
    portMapper: portmapper.New(d.config.UserlandProxyPath),
    bridge:     bridgeIface,
    driver:     d,
}

d.Lock()
d.networks[config.ID] = network
d.Unlock()
```

如果获取的 bridgeInterface 中不存在有效网桥设备，则将创建设备、sysctl 方法加入设置队列；如果使用 docker0，仅将 sysctl 方法加入设置队列

```go
bridgeAlreadyExists := bridgeIface.exists()
if !bridgeAlreadyExists {
    bridgeSetup.queueStep(setupDevice)
    bridgeSetup.queueStep(setupDefaultSysctl)
}

// For the default bridge, set expected sysctls
if config.DefaultBridge {
    bridgeSetup.queueStep(setupDefaultSysctl)
}
```

根据配置文件参数，将对应的设置方法加入设置队列

```go
for _, step := range []struct {
        Condition bool
        Fn        setupStep
}{
    // Enable IPv6 on the bridge if required. We do this even for a
    // previously  existing bridge, as it may be here from a previous
    // installation where IPv6 wasn't supported yet and needs to be
    // assigned an IPv6 link-local address.
    {config.EnableIPv6, setupBridgeIPv6},

    // We ensure that the bridge has the expectedIPv4 and IPv6 addresses in
    // the case of a previously existing device.
    {bridgeAlreadyExists && !config.InhibitIPv4, setupVerifyAndReconcile},

    // Enable IPv6 Forwarding
    {enableIPv6Forwarding, setupIPv6Forwarding},

    // Setup Loopback Addresses Routing
    {!d.config.EnableUserlandProxy, setupLoopbackAddressesRouting},

    // Setup IPTables.
    {d.config.EnableIPTables, network.setupIPTables},

    //We want to track firewalld configuration so that
    //if it is started/reloaded, the rules can be applied correctly
    {d.config.EnableIPTables, network.setupFirewalld},

    // Setup DefaultGatewayIPv4
    {config.DefaultGatewayIPv4 != nil, setupGatewayIPv4},

    // Setup DefaultGatewayIPv6
    {config.DefaultGatewayIPv6 != nil, setupGatewayIPv6},

    // Add inter-network communication rules.
    {d.config.EnableIPTables, setupNetworkIsolationRules},

    //Configure bridge networking filtering if ICC is off and IP tables are enabled
    {!config.EnableICC && d.config.EnableIPTables, setupBridgeNetFiltering},
} {
    if step.Condition {
        bridgeSetup.queueStep(step.Fn)
    }
}
```

将设备启动设置方法加入设置队列，并返回执行结果

```go
bridgeSetup.queueStep(setupDeviceUp)
return bridgeSetup.apply()
```

#### Setup Device

创建 netlink.Bridge 结构体，LinkAttrs 中使用配置中的 BridgeName，然后，使用 netlink 方法创建网桥设备，如果需要设置 MAC 则随机生成 MAC 地址。

```go
func setupDevice(config *networkConfiguration, i *bridgeInterface) error {
    var setMac bool

    // We only attempt to create the bridge when the requested device name is
    // the default one.
    if config.BridgeName != DefaultBridgeName && config.DefaultBridge {
        return NonDefaultBridgeExistError(config.BridgeName)
    }

    // Set the bridgeInterface netlink.Bridge.
    i.Link = &netlink.Bridge{
        LinkAttrs: netlink.LinkAttrs{
            Name: config.BridgeName,
        },
    }

    // Only set the bridge's MAC address if the kernel version is > 3.3, as it
    // was not supported before that.
    kv, err := kernel.GetKernelVersion()
    if err != nil {
        logrus.Errorf("Failed to check kernel versions: %v. Will not assign a MAC address to the bridge interface", err)
    } else {
        setMac = kv.Kernel > 3 || (kv.Kernel == 3 && kv.Major >= 3)
    }

    if setMac {
        hwAddr := netutils.GenerateRandomMAC()
        i.Link.Attrs().HardwareAddr = hwAddr
        logrus.Debugf("Setting bridge mac address to %s", hwAddr)
    }

    if err = i.nlh.LinkAdd(i.Link); err != nil {
        logrus.Debugf("Failed to create bridge %s via netlink. Trying ioctl", config.BridgeName)
        return ioctlCreateBridge(config.BridgeName, setMac)
    }

    return err
}
```

Bridge 设备创建、配置等，最终均通过 netlink 接口完成。

#### Networking Configuration

* System Control
  * /proc/sys/net/ipv6/conf/BridgeName/accept\_ra -&gt; 0：不接受路由建议
  * /proc/sys/net/ipv4/conf/_**BridgeName**_/route\_localnet -&gt; 1：将外部流量重定向至 loopback，需要配合 iptables 使用

#### IPTABLES

* INTERNAL
  * filter
    * DOCKER-ISOLATION-STAGE-1 -i _**BridgeInterface**_ ! -d _**Network**_ -j DROP
    * DOCKER-ISOLATION-STAGE-1 -o _**BridgeInterface**_ ! -s _**Network**_ -j DROP
* NON INTERNAL
  * nat
    * DOCKER -t nat -i _**BridgeInterface**_ -j RETURN
  * filter
    * FORWARD -i _**BridgeInterface**_ ! -o _**BridgeInterface**_ -j ACCEPT
  * HOST IP != nil
    * nat
      * POSTROUTING -t nat -s _**BridgeSubnet**_ ! -o _**BridgeInterface**_ -j SNAT --to-source _**HOSTIP**_
      * POSTROUTING -t nat -m addrtype --src-type LOCAL -o _**BridgeInterface**_ -j SNAT --to-source _**HOSTIP**_
  * HOST IP == nil
    * nat
      * POSTROUTING -t nat -s _**BridgeSubnet**_ ! -o _**BridgeInterface**_ -j MASQUERADE
      * POSTROUTING -t nat -m addrtype --src-type LOCAL -o _**BridgeInterface**_ -j MASQUERADE
  * Inter Container Communication Enabled
    * filter
      * FORWARD -i _**BridgeInterface**_ -o _**\_\_BridgeInterface**_ -j ACCEPT
  * Inter Container Communication Disabled
    * filter
      * FORWARD -i _**BridgeInterface**_ -o _**\_\_BridgeInterface**_ -j DROP
  * nat
    * PREROUTING -m addrtype --dst-type LOCAL -j DOCKER 
    * OUTPUT  -m addrtype --dst-type LOCAL -j DOCKER 
  * filter
    * FORWARD -o _**BridgeInterface**_ -j DOCKER
    * FORWARD -o _**BridgeInterface**_ -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
  * filter
    * -I FORWARD -j DOCKER-ISOLATION-STAGE-1

### Create Endpoint

![docker-network-bridge-network.svg](../.gitbook/assets/8%20%282%29.jpeg)  
全局有一个默认 Bridge 设备 docker0，每个 Container 有自己独立的网络协议栈，容器网络和通过 veth 对与 Bridge 设备互通。  
同一节点上不同 Container 间，通过 ARP 协议，即可进行 3 层通信；Container 出 Node 网络可以通过默认网关设备 docker0，再经过 IPTABLES 重定向至 eth0。

