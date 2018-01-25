# UPnP设备和控制点体系结构

## UPnP寻址
因为所有UPnP通信都是通过Internet协议（IP）进行的，所以目标设备必须先获得一个IP地址，然后才能加入支持UPnP的网络。

UPnP设备可以使用动态主机配置协议（DHCP）从DHCP服务器检索IP地址，如果网络没有DHCP服务器，UPnP设备可以使用自动IP殉职来获取IP地址。

通过使用IP地址，控制点可以联系与控制点位于同一子网上的UPnP设备。

## UPnP发现
UPnP发现协议允许设备通告控制网络上可用的点，还允许控制点查找哪些设备可以被控制。

当设备在网络上可用时，它通过多播简单服务发现协议（SSDP）NOTIFY消息将其嵌入式设备和服务通告给网络上的控制点。

当控制点在网络上可用时，它通过使用基于UDP（HTTPMU）请求的HTTP多播来组播`SSDP M_SEARCH`消息来搜索可用设备和服务。搜索消息包含指定控制点正在搜索的设备或服务的类型的资格列表。

如果网络上的设备包含符合搜索标准的嵌入式设备或服务，则它将使用包含SSDP标头的UDP响应来响应控制点。每个响应都包含一个URL，控制点可以从该URL中检索该设备的设备描述信息。

## UPnP设备说明
控制点为搜索请求收到的响应包含提供设备功能描述的URL。要检索单个设备的描述，控制点向此URL发出HTTP GET请求。

当一个设备接收到设备描述信息的请求时，它会在消息体中回复一个包含设备描述的HTTP响应。UPnP设备描述信息包括：

包含多个设备特定数据的XML文档。
所有嵌套设备的定义。
设备支持的所有服务的列表，包括状态变量和操作。
发现消息包含设备描述文档的URL; 类似地，在SCPDURL元素的情况下，设备描述文档包含用于服务描述文档的URL。服务描述文档包含有关操作和状态变量的信息。

以下XML是一个示例XML设备描述文档。C Device Host API和COM Device Host API都使用这样的设备描述文档。
```xml
<?xml version="1.0"?>
<root xmlns="urn:schemas-upnp-org:device-1-0">
  <specVersion>
    <major>1</major>
    <minor>0</minor>
  </specVersion>

  <device>
    <deviceType>urn:schemas-upnp-org:device.lighting.2</deviceType>
    <friendlyName>Dan's UPnP Device Host Test</friendlyName>
    <manufacturer>Microsoft</manufacturer>
    <manufacturerURL>http://www.microsoft.com/</manufacturerURL>
    <modelDescription>Dan's UPnP-X10 Light and Dimmer control</modelDescription>
    <modelName>X-10L1</modelName>
    <modelNumber>L1</modelNumber>
    <modelURL>http://www.microsoft.com/</modelURL>
    <serialNumber>0000001</serialNumber>
    <UDN>DummyUDN</UDN>
    <UPC>00000-00001</UPC>

    <iconList>
      <icon>
        <mimetype>image/png</mimetype>
        <width>16</width>
        <height>16</height>
        <depth>2</depth>
        <url>icon.png</url>
      </icon>
    </iconList>
    <serviceList>
      <service>
        <serviceType>urn:schemas-upnp-org:service:pwrdim:2</serviceType>
        <serviceId>upnp:id:pwrdim</serviceId>
        <controlURL></controlURL>
        <eventSubURL></eventSubURL>
        <SCPDURL>SampleSCPD.xml</SCPDURL>
      </service>
    </serviceList>

    <presentationURL>sample.html</presentationURL>

  </device>
</root>
```

## UPnP控制
控制点获知设备及其服务后，可以使用这些服务调用动作并查询状态变量值。

要调用特定的操作：

控制点向设备的服务发送包含SOAP格式内容的POST控制消息。控制消息包含供应商特定的信息，操作名称，参数名称和由UPnP论坛定义的变量。

该服务回复一个HTTP响应，通知控制点请求的状态。如果请求可以在30秒内完成，包括预期的传输时间，则响应包含请求的结果。如果请求花费的时间超过30秒，则设备返回一个没有完整数据的响应，然后在请求完成时发送一个事件。

控制点也可以查询设备的服务状态变量。除了SOAP请求和回复的内容使用预定义的QueryStateVariable名称，而不是由服务定义的特定操作名称之外，在这种情况下控制点和设备之间的通信与动作调用相同。

## UPnP事件
通过注册事件，UPnP控制点可以在状态变量发生变化时收到通知。在这种情况下，一个变量是一个变量，当控制点的值发生变化时，这个变量就发送给控制点。非均匀变量不发送通知。

控制点使用通过HTTP和TCP传输的通用事件通知架构（GENA）消息来订阅事件。UPnP事件不使用SOAP。

订阅特定事件：

控制点向服务发送GENA SUBSCRIBE消息。
该服务接受或拒绝订阅。
如果服务接受订阅，它将以持续时间值进行响应。
控制点必须在持续时间值表示的时间通过之前定期更新其订阅。
如果订阅不更新，服务取消订阅，不再发送事件。
当控制点订阅特定事件时，设备将所有变量的完整状态发送到控制点。提供此初始状态后，如果发生特定变量的任何更改，则设备将向控制点发送更新。

一个控制点可以订阅任何数量的变量。控制点可以在不再需要时取消订阅。

## UPnP用户界面
在启用UPnP的网络中，控制点可以通过UPnP设备返回的呈现HTML页面（或一系列页面）来控制设备或查看其状态。控制页面使人们可以通过设备提供的界面查看和控制设备，而不是通过控制点构建和提供的界面。

控制页面不是必需的; 如果设备没有演示页面，仍然可以通过标准的UPnP控制消息进行控制。

如果设备支持演示文稿页面，则设备描述文档将在<presentationURL>标签中包含控制页面的URL。（<presentationURL>标签是必需的;如果设备没有控制页面，标签应该是空的。）

要检索控制文稿页面，控制点将使用控制文稿URL提交HTTP GET请求。



## UPnP 设备控制协议

UPnP社区为设备发现，描述和控制等操作定义了一组标准操作和一组标准通信协议。这些操作提供了构建实际UPnP设备的构建模块。UPnP设备和控制点体系结构中详细介绍了这些概念。

标准操作不定义设备实现的特定服务和操作。例如，控制点可以使用设备发现来查找所有UPnP设备和设备描述文档，以准确确定设备支持的服务和操作。但是，如果在相同类型的设备上的服务和操作之间没有共同的假设，则控制点不能对其控制的设备的类型做出假设。这意味着控制点必须与完全独立的实体进行相同类型的设备的交互。

为了纠正这个问题，UPnP论坛也定义了**DCP**。DCP是一组特定的命名服务，每个服务都有特定的命名操作和状态变量。由于服务，动作和状态变量具有通用的名称和语义，因此控制点可以以通用的方式与所有相同类型的设备进行交互。

例如，UPnP音频/视频（AV）DCP定义用于呈现和服务媒体内容的设备和服务。了解UPnP AV DCP的UPnP控制点可以为网络上的所有UPnP AV渲染器和服务器设备提供通用的控制机制。

除了支持标准的UPnP操作之外，还包括以下UPnP DCP的附加功能：

- 音频/视频（AV）。包含UPnP AV框架。UPnP AV框架是一个C ++框架，协助开发人员创建UPnP AV DCP控制点和设备。有关UPnP AV框架的更多信息，请参阅使用UPnP AV框架。
- 互联网网关设备（IGD）。Windows CE包含IGD DCP的示例实现。有关此实现的更多信息，请参阅实现Internet网关设备架构和UPnP Internet网关设备架构示例。
  有关其他UPnP DCP的更多信息，请参阅此[UPnP网站](http://go.microsoft.com/fwlink/?linkid=11903)。您可以使用通用的UPnP控制点和设备主机API来实现与任何DCP协同工作的控制点和设备。



## 使用UpnP进行内网穿透

** 使用UPnP进行内网穿透的前提是集成NAT功能的网关要有公网IP**，也就是只有在一级NAT时才可以使用NAT进行内网穿透。

UPnP是开放的设备互联协议，基于TCP/IP且不需要驱动。开启UPnP功能的NAT设备，内网终端可以让NAT网关做自动端口映射。UPnP协议簇实现互联互通，就是将内网终端的外网地址广播公告出去。UPnP可以穿透UDP/TCP NAT和对称型NAT，穿透效率很高，但应用场景有限。在串联多级NAT设备的情况下，需要每级都打开UPnP功能请求自动端口映射，外网服务器才能获知内网终端的公网地址。但事实上，外层NAT设备是不可预知和控制的。UpnP技术属于对NAT设备干预的一种穿越NAT技术，它允许终端控制NAT，需要NAT设备支持UPnP协议。故针对UPnP的穿透方法目前公网IP日益缺乏的情况下应用场景有限。

UPnP主要通过用户控制点向NAT设备发送控制信息添加端口映射，端口添加成功后向服务器注册自己的端口信息，这样设备即可以通过服务器得知控制端的端口信息从而实现NAT穿越。UPnP的优点在于不用对现有设备进行改造。但它有一个明确的要求，即集成NAT功能的网关或路由器需要支持UPnP功能并且UPnP是打开的。目前大多数大型网关或路由器都支持UPnP解决方案并且配置简单，因而大多数P2P应用都采用UPnP技术解决NAT穿透问题。



