# ZeroNet 节点与节点间通信
在获取了节点信息后，节点之间获取网站，发布更新，验证更新等这一系列操作是通过ZeroNet自带协议完成的。每条信息都利用`Msgpack`进行编码.

ZeroNet节点之间主要有三种通信会产生通信:

- 一个节点从另一个节点下载站点

- 一个站点向其节点列表中的节点推送网站更新

- 两个节点之间互相交换节点列表

## ZeroNet 通信格式

每条请求包括三个参数`cmd`,`req_id`,`params`。每条请求都始于向目标节点发送`Handshake`。

- cmd：请求命令

- req_id：请求的唯一标识符（简单，递增的每个连接的随机数），客户端在回复命令时必须包含这个标识符。

- params：请求的参数

-示例请求：` {"cmd": "getFile", "req_id": 1, "params:" {"site": "1EU...", "inner_path": "content.json", "location": 0}}`

-响应示例：` {"cmd": "response", "to": 1, "body": "content.json content", "location": 1132, "size": 1132}`

- 示例错误响应：` {"cmd": "response", "to": 1, "error": "Unknown site"}`

### 握手

每个连接都通过向目标网络地址发送请求的握手开始：

| Parameter       | Description         |
| :-------------- | :------------------ |
| crypt           | 空/无，仅用于响应           |
| crypt_supported | 客户端支持的一系列连接加密方法     |
| fileserver_port | 客户端的文件服务器端口         |
| onion           | 仅用于tor）客户端的洋葱地址     |
| protocol        | 客户端使用的协议版本（v1或v2）   |
| port_opened     | 客户端的客户端端口打开状态       |
| peer_id         | （不用于tor）客户端的peer_id |
| rev             | 客户的修订号              |
| version         | 客户端的版本              |
| target_ip       | 服务器的网络地址            |

目标基于socket初始化加密crypt_supported，然后返回：

| Return key      | Description                              |
| --------------- | ---------------------------------------- |
| crypt           | 要使用的加密                    |
| crypt_supported | 服务器支持的一系列连接加密方法 |
| fileserver_port | 服务器的文件服务器端口            |
| onion           | （仅用于tor）服务器的洋葱地址 |
| protocol        | 服务器使用的协议版本（v1或v2） |
| port_opened     | 服务器的客户端端口打开状态    |
| peer_id         | (不用于tor)服务器的peer_id   |
| rev             | 服务器的修订号             |
| target_ip       | 客户的网络地址             |

>注意：在.onion连接上不使用加密，因为Tor网络默认提供传输安全性。 注意：如果您可以假定远程客户端支持，也可以在握手之前隐式初始化SSL。

我们举一个发送握手的例子：

```python
{
  "cmd": "handshake",
  "req_id": 0,
  "params": {
    "crypt": None,
    "crypt_supported": ["tls-rsa"],
    "fileserver_port": 15441,
    "onion": "zp2ynpztyxj2kw7x",
    "protocol": "v2",
    "port_opened": True,
    "peer_id": "-ZN0056-DMK3XX30mOrw",
    "rev": 2122,
    "target_ip": "192.168.1.13",
    "version": "0.5.6"
  }
}
```

返回:
```python
{
 "protocol": "v2",
 "onion": "boot3rdez4rzn36x",
 "to": 0,
 "crypt": None,
 "cmd": "response",
 "rev": 2092,
 "crypt_supported": [],
 "target_ip": "zp2ynpztyxj2kw7x.onion",
 "version": "0.5.5",
 "fileserver_port": 15441,
 "port_opened": False,
 "peer_id": ""
}
```

通过握手这个过程，通信的节点双方知晓了双方的状态，接下来就可以利用双方都接受的通信过程来进行通信。

### cmd包含的请求命令
在通信节点双方完成握手之后下一步就是发送命令(cmd)了，ZeroNet定义了以下命令：

|命令(cmd) | 意思|
|---|---|
|getFile|从客户端请求一个文件|
|streamFile|从客户端流式传输文件|
|ping|检查客户是否还活着|
|pex|与节点交换节点。对等打包到6个字节（使用inet_ntoa + 2byte作为端口的4字节IP）|
|update|更新站点文件|
|listModified|列出自给定参数后修改的content.json文件。它用来获取网站的用户提交的内容。|
|getHashfield|获取客户端下载的可选文件ID|
|setHashfield|设置请求者客户端具有的可选文件ID列表|
|findHashIds|查询客户端是否知道具有请求的hash_ids的对等体|
|checkport|检查对方的请求端口|
|getPieceFields|返回客户端为该网站在字典中的所有大文件片段|
|setPieceFields|设置该网站的客户的片段|


## ZeroNet 通信加密协议

## ZeroNet 通信编码协议

##一次完整的通信协议
首先我们从建站开始，按照ZeroNet的说明建立了一个新的空站点`1KhbkxNXzJW9h3rxFXm5AmAwpQ6DifHumY`，网站内置的是初始的节点信息。我们的ZeroNet客户端会通知Tracker记录该站点。接下来，我们在局域网内的另外一台机器上访问此空战点，先从Tracker获取该站点的peer表，然后连接peer进行P2P下载。在拥有Peer表之后，以后该Peer 会之间和Peer表中的其他节点通信更新网站数据和获取新节点列表更新。

在P2P部分，我们首先发送Handshark 命令，完成两个Peer之间的握手协议（ZeroNet自带），通过抓包，通信过程如下,以一个更新过程为例：

对于数据更新操作是通过TCP连接，命令格式为JSON，命令和数据由Msgpack进行编码。先进行握手，文件内容，更新时间，支持的加密协议，作者签名等验证，通过’tls-rsa‘加密进行传输。

`192.168.204.1 --> 192.168.204.130`

```json
..protocol.v2.fileserver_port.<Q.target_ip.192.168.204.1.port_opened..crypt..cmd.response.rev..j.to..version.0.6.0.crypt_supported..tls-rsa.peer_id.-ZN0060-YRkWn3hqTQwB..cmd.response.to..modified_files..content.json.Z].^..to..cmd.response.ok.."Thanks, file content.json updated!..cmd.streamFile.params..inner_path.services.html.read_bytes......file_size.#..site.."1KhbkxNXzJW9h3rxFXm5AmAwpQ6DifHumY.location..req_id...cmd.update.params..inner_path.content.json.body.@.{
 "address": "1KhbkxNXzJW9h3rxFXm5AmAwpQ6DifHumY",
 "address_index": 49513077,
 "background-color": "#FFF",
 "clone_root": "template-new",
 "cloned_from": "1HeLLo4uzjaLetFx6NH3PMwFP3qbRbTf3D",
 "description": "\u554a\u624b\u52a8\u9600\u6492\u65e6\u98de\u6d12\u5730\u65b9\u6492\u65e6\u53d1\u5c04\u70b9\u53d1\u6492\u65e6\u53d1\u5c04\u70b9\u53d1\u5c04\u70b9\u53d1",
 "files": {
 .......
 },
 "ignore": "",
 "inner_path": "content.json",
 "modified": 1516091926,
 "postmessage_nonce_security": true,
 "signers_sign": "HFvzFf5QvXl4I6cvKwTw040pQfBmGdS8MpEw/yBeJOrxBQndTvfh39p+8bdlpwEDdraf4g3cHvOQbDUxwC+Kyfc=",
 "signs": {
  "1KhbkxNXzJW9h3rxFXm5AmAwpQ6DifHumY": "GwDe6jV4UFaisemgNY+jOUNjSooJB08BlCbXEPQq2/U5egmKvIzLUlDJTGkDXqmfoJXIYOWW0Ir0yB7XeukQSsI="
 },
 "signs_required": 1,
 "title": "my new site\u554a\u624b\u52a8\u9600\u624b\u52a8\u9600\u624b\u52a8\u9600\u6492\u65e6",
 "translate": ["js/all.js"],
 "zeronet_version": "0.6.0"
}.site.."1KhbkxNXzJW9h3rxFXm5AmAwpQ6DifHumY.diffs..req_id.
```

`192.168.204.130 --> 192.168.204.1`

```json
..protocol.v2.fileserver_port.<Q.target_ip.192.168.204.1.port_opened..crypt..cmd.response.rev..j.to..version.0.6.0.crypt_supported..tls-rsa.peer_id.-ZN0060-YRkWn3hqTQwB..cmd.response.to..modified_files..content.json.Z].^..to..cmd.response.ok.."Thanks, file content.json updated!..cmd.streamFile.params..inner_path.services.html.read_bytes......file_size.#..site.."1KhbkxNXzJW9h3rxFXm5AmAwpQ6DifHumY.location..req_id...cmd.update.params..inner_path.content.json.body.@.{
 "address": "1KhbkxNXzJW9h3rxFXm5AmAwpQ6DifHumY",
 "address_index": 49513077,
 "background-color": "#FFF",
 "clone_root": "template-new",
 "cloned_from": "1HeLLo4uzjaLetFx6NH3PMwFP3qbRbTf3D",
 "description": "\u554a\u624b\u52a8\u9600\u6492\u65e6\u98de\u6d12\u5730\u65b9\u6492\u65e6\u53d1\u5c04\u70b9\u53d1\u6492\u65e6\u53d1\u5c04\u70b9\u53d1\u5c04\u70b9\u53d1",
 "files": {
  ......
  }
 },
 "ignore": "",
 "inner_path": "content.json",
 "modified": 1516091926,
 "postmessage_nonce_security": true,
 "signers_sign": "HFvzFf5QvXl4I6cvKwTw040pQfBmGdS8MpEw/yBeJOrxBQndTvfh39p+8bdlpwEDdraf4g3cHvOQbDUxwC+Kyfc=",
 "signs": {
  "1KhbkxNXzJW9h3rxFXm5AmAwpQ6DifHumY": "GwDe6jV4UFaisemgNY+jOUNjSooJB08BlCbXEPQq2/U5egmKvIzLUlDJTGkDXqmfoJXIYOWW0Ir0yB7XeukQSsI="
 },
 "signs_required": 1,
 "title": "my new site\u554a\u624b\u52a8\u9600\u624b\u52a8\u9600\u624b\u52a8\u9600\u6492\u65e6",
 "translate": ["js/all.js"],
 "zeronet_version": "0.6.0"
}.site.."1KhbkxNXzJW9h3rxFXm5AmAwpQ6DifHumY.diffs..req_id.
```

```
..protocol.v2.fileserver_port.<Q.target_ip.192.168.204.1.port_opened..crypt..cmd.response.rev..j.to..version.0.6.0.crypt_supported..tls-rsa.peer_id.-ZN0060-YRkWn3hqTQwB..cmd.response.to..modified_files..content.json.Z]....cmd.response.to..modified_files..content.json.Z]....to..cmd.response..to..cmd.response..to..cmd.response
```

经过验证，ZeroNet Peer 之间的通信在局域网内是正常的，数据在Peer 传输完成后Ui会显示站点已更新，刷新后可以看见更新。ZeroNet的15441是默认的站点发布端口，也是站点接收端口，只要存在数据库中的站点Peer端口间连接正常，数据就可以在Peer间传输。

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

\### NAT穿透

一种称为互联网网关设备协议（IGD协议）的NAT穿透解决方案是通过UPnP实现的。许多路由器和防火墙将自己公开为Internet网管设备，运行任何本地UPnP控制点执行各种操作，包括检索设备的外部IP地址，枚举现有的端口映射已经添加或删除端口映射。通过添加端口映射，IGD后面的UPnP控制器可以启用IGD从外部地址到内部客户端的遍历。

## IGD

![](https://upload.wikimedia.org/wikipedia/commons/3/36/UPnP_discovery_phase.jpg)

**互联网网关设备协议**（英语：Internet Gateway Device，缩写**IGD**）标准设备控制协议（Standardized Device Control Protocol）是一种在（NAT）环境中映射的协议，受部分支持NAT的支持。它是一种常见的自动配置，并是[ISO](https://zh.wikipedia.org/wiki/%E5%9C%8B%E9%9A%9B%E6%A8%99%E6%BA%96%E5%8C%96%E7%B5%84%E7%B9%94)/[IEC](https://zh.wikipedia.org/wiki/%E5%9B%BD%E9%99%85%E7%94%B5%E5%B7%A5%E5%A7%94%E5%91%98%E4%BC%9A)标准的一部分，但不是一项标准。

使用点对点网络网络、多媒体游戏或远程协助程序的应用程序需要一种穿透家庭和商用网关进行通信的方法。在没有IGD支持时，手动配置网关以允许流量通过是一个容易出错且费时费力的过程。UPnP则带来了另一个NAT穿透解决方案。

IGD可以轻松执行如下操作：

- 获知公网（外部）IP地址

- 请求一个新的公网IP地址

- 列举现有的端口映射

- 添加和移除端口映射

- 给映射分配租赁时间

主机可以允许通过简单服务发现协议（SSDP）寻找网络上的可用设备，然后在简单对象访问协议（SOAP）的帮助下进行控制。此种寻找请求通过超文本传输协议（HTTP）和通信端口1900发送到[多播地址](https://zh.wikipedia.org/w/index.php?title=%E5%A4%9A%E6%92%AD%E5%9C%B0%E5%9D%80&action=edit&redlink=1)239.255.255.250，示例如下：

```

M-SEARCH * HTTP/1.1

Host:239.255.255.250:1900

ST:urn:schemas-upnp-org:device:InternetGatewayDevice:1

Man:"ssdp:discover"

MX:3

```