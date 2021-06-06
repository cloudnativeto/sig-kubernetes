# Docker Networking

## Design

![image.png](../.gitbook/assets/1%20%2817%29.jpeg)

* Sandbox: The protocol stack can contain multiple endpoints, which can be implemented by Namespace, Jail, etc.
* Endpoint: Connect Sandbox with Network
* Network: A collection of Endpoints that can communicate directly, which can be implemented using Bridge, VLAN, etc.

## Docker Architecture

![docker-network-arch.svg](../.gitbook/assets/2%20%2811%29.jpeg)

## Network Controller

![docker-network-network-controller.svg](../.gitbook/assets/3%20%2810%29.jpeg)

### Initialize Network Controllers

Docker Daemon Manages the available NetWorkController. When launching Daemon, all available NetWorkController under the current operating system will be created. 

On Unix Operating system: with daemon\_unix.go as an example, create None, Host, Bridge network controller.

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

![docker-network-network-controller-impl.svg](../.gitbook/assets/4%20%2810%29.jpeg)

  
The controller is the implementation of NetworkController in libnetwork. In this picture, the controller uses a registry map to distinguish network with different types, then use the Driver to create Network and Endpoint, attach the Endpoint to Sandbox or remove them from Sandbox.

The Container uses SandboxID and SandboxKey to find Sandbox. At the same time, Sandbox use containerID to determine which Container it belongs to.      

## OS Layer Sandbox

### Namespace

![docker-network-sandbox.svg](../.gitbook/assets/5%20%285%29.jpeg)

As indicated by docker-network-sandbox.svg, not all the functions are listed, but only the functions which divide the border of Sandbox. The Namespace implementation will be an example of analytic.  
netlink provides the functions like route, interface. The Namespace could get netlink as below code. Attention:  netlink configure only work in Namespace.

```go
func GetFromPath(path string) (NsHandle, error) {
    fd, err := syscall.Open(path, syscall.O_RDONLY, 0)
    if err != nil {
        return -1, err
    }
    return NsHandle(fd), nil
}
```

GetFromPath would return NsHandle structure, then NsHandle could be used in below methods to create specific SocketHandle.

```
func NewHandleAt(ns netns.NsHandle, nlFamilies ...int) (*Handle, error) {
```

```go
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

![docker-network-add-interface.svg](../.gitbook/assets/6%20%284%29.jpeg)

## Bridge Network

![docker-network-bridge-overview.svg](../.gitbook/assets/7%20%285%29.jpeg)

### Create Network

According to the BridgeName value of the config file\(networkConfiguration\), attempt to find an existing bridge named with the specified name. If not, use the default Bridge -- docker0.

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

Create and set network handler in driver

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

If bridgeInterface exists the valid bridge device, the device and sysctl methods would be added to the queue; if already exists, just add sysctl methods.

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

Add a corresponding setting method to the setting queue according to the configuration file parameters.

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

Add the device start setting method to the setup queue and return to the execution result.

```go
bridgeSetup.queueStep(setupDeviceUp)
return bridgeSetup.apply()
```

#### Setup Device

Create a NetLink.bridge structure, using BridGename in the configuration to create LinkAttrs, then create a bridge device using the NetLink method. If need to set MAC, randomly generates the MAC address.

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

So netlink could create and configure Bridge devices. 

#### Networking Configuration

* System Control
  * /proc/sys/net/ipv6/conf/BridgeName/accept\_ra -&gt; 0：Routing suggestions are not accepted
  * /proc/sys/net/ipv4/conf/_**BridgeName**_/route\_localnet -&gt; 1：Redirect external traffic to loopback, need to be used with iptables

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

![docker-network-bridge-network.svg](../.gitbook/assets/8%20%284%29.jpeg)

  
There is a default Bridge device docker0 globally, and each Container has its own independent network protocol stack. Container network and Bridge device communication via Veth pairs  
Different Container on the same node, 3-layer communication can be performed through the ARP protocol; when the Container traffic out of the Node network, should be redirected by the default gateway device Docker0, and then redirect to Eth0.

