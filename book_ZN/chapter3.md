# ZeroNet 节点与节点间通信

从跟踪器或者通过节点交换（PEX）获得节点列表后，通过Msgpack解析出节点列表中的内容。

## ZeroNet 通信格式


## ZeroNet 通信加密协议

## ZeroNet 通信编码协议

## UPnP

**通用即插即用**（Universal Plug and Play，简称UPnP）是由“通用即插即用论坛”推广的一套网络协议。该协议的目标是使家庭网络（数据共享、通信和娱乐）和网络中各种设备能够相互无缝连接，并简化相关网络的实现。相关的UPnP实现可以看这篇文章:[WANIPConnection:1 Service Template Version 1.01](http://www.upnp.org/specs/gw/UPnP-gw-WANIPConnection-v1-Service.pdf)
UPnP体系允许PC间的点对点连接、网际互连和无线设备。它是一种基于TCP/IP、UDP和HTTP的分布式、开放体系。UPnP使得任意两个设备能在LAN控制设备的管理下相互通信。其特性包括：

* 传输介质和设备独立。UPnP技术可以应用在许多媒体上，包括电话线、电线（电力线通信PLC）、以太网、红外通信技术（IrDA）、无线电（Wi-Fi，蓝牙）和Firewire（1394）。无需任务设备驱动；而是采用共同的协议。

* 用户界面（UI）控制。UPnP技术使得设备厂商可以通过网页浏览器来控制设备并进行交互。
  操作系统和程序语言独立。任何操作系统和程序语言均可以用于构建UPnP产品。UPnP并没有设定或限制运行于控制设备上的应用程序API；OS厂商可以创建满足他们客户需求的API。UPnP使得厂商可以像开发常规应用程序一样来控制设备UI和交互。

* 基于因特网技术。UPnP构建于IP、TCP、UDP、HTTP，和XML等许多协议之上。

* 编程控制。UPnP体系同时支持常规应用程序编程控制。

*扩展性。每个UPnP设备都可以有构建于基本体系之上、与具体设备相关的服务。

UPnP支持零配置，"看不见的网络"和自动检测；任何设备能自动加入一个网络，获取一个IP地址，宣布自己的名字，根据请求检查自身功能以及检测出其它设备和它们的功能。DHCP和DNS服务是可选的，并只有它们在网络上存在的时候才会使用。设备可以自动离开网络而不会遗留下任何不需要的状态信息。

UPnP的基础是IP地址解析。每一个设备都应当有一个DHCP客户端并在连入网络的时候自动搜索DHCP服务。如果没有找到DHCP服务，也就是说网络是缺乏管理状态，那么设备必须给自己设定一个地址。如果在和DHCP服务器交互的过程中，设备获得了一个域名（比如通过DNS服务器或者DNS传递），那么它应当在接下来的网络操作中使用这个域名；否则，设备应当使用它的IP地址。

### NAT穿透
一种称为互联网网关设备协议（IGD协议）的NAT穿透解决方案是通过UPnP实现的。许多路由器和防火墙将自己公开为Internet网管设备，运行任何本地UPnP控制点执行各种操作，包括检索设备的外部IP地址，枚举现有的端口映射已经添加或删除端口映射。通过添加端口映射，IGD后面的UPnP控制器可以启用IGD从外部地址到内部客户端的遍历。

## IGD

![](https://upload.wikimedia.org/wikipedia/commons/3/36/UPnP_discovery_phase.jpg)

**互联网网关设备协议**（英语：Internet Gateway Device，缩写**IGD**）标准设备控制协议（Standardized Device Control Protocol）是一种在（NAT）环境中映射的协议，受部分支持NAT的支持。它是一种常见的自动配置，并是[ISO](https://zh.wikipedia.org/wiki/%E5%9C%8B%E9%9A%9B%E6%A8%99%E6%BA%96%E5%8C%96%E7%B5%84%E7%B9%94)/[IEC](https://zh.wikipedia.org/wiki/%E5%9B%BD%E9%99%85%E7%94%B5%E5%B7%A5%E5%A7%94%E5%91%98%E4%BC%9A)标准的一部分，但不是一项标准。

使用点对点网络网络、多媒体游戏或远程协助程序的应用程序需要一种穿透家庭和商用网关进行通信的方法。在没有IGD支持时，手动配置网关以允许流量通过是一个容易出错且费时费力的过程。UPnP则带来了另一个NAT穿透解决方案。

IGD可以轻松执行如下操作：

-   获知公网（外部）IP地址
-   请求一个新的公网IP地址
-   列举现有的端口映射
-   添加和移除端口映射
-   给映射分配租赁时间

主机可以允许通过简单服务发现协议（SSDP）寻找网络上的可用设备，然后在简单对象访问协议（SOAP）的帮助下进行控制。此种寻找请求通过超文本传输协议（HTTP）和通信端口1900发送到[多播地址](https://zh.wikipedia.org/w/index.php?title=%E5%A4%9A%E6%92%AD%E5%9C%B0%E5%9D%80&action=edit&redlink=1)239.255.255.250，示例如下：

```
M-SEARCH * HTTP/1.1
Host:239.255.255.250:1900
ST:urn:schemas-upnp-org:device:InternetGatewayDevice:1
Man:"ssdp:discover"
MX:3
```


