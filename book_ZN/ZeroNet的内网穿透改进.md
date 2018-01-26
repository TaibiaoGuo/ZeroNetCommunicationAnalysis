# ZeroNet的内网穿透改进

## ZeoNet 的内网穿透实现



## ZeroNet 内网穿透的局限性



## 其他内网穿透列举

### STUN 


STUN是一种网络协议，它允许位于NAT（或多重NAT）后的客户端找出自己的公网地址，查出自己位于哪种类型的NAT之后以及NAT为某一个本地端口所绑定的Internet端端口。这些信息被用来在两个同时处于NAT路由器之后的主机之间建立UDP通信。该协议由RFC 5389定义。STUN由三部分组成：STUN客户端、STUN服务器端、NAT路由器。STUN服务端部署在一台有着两个公网IP的服务器上。大概的结构参考下图。STUN客户端通过向服务器端发送不同的消息类型，根据服务器端不同的响应来做出相应的判断，一旦客户端得知了Internet端的UDP端口，通信就可以开始了。

![img](https://blog-10039692.file.myqcloud.com/1505808494825_839_1505808494993.png)

STUN协议定义了三类测试过程来检测NAT类型。

**Test1：**STUN Client通过端口{IP-C1:Port-C1}向STUN Server{IP-S1:Port-S1}发送一个Binding Request（没有设置任何属性）。STUN Server收到该请求后，通过端口{IP-S1:Port-S1}把它所看到的STUN Client的IP和端口{IP-M1,Port-M1}作为Binding Response的内容回送给STUN Client。Test1#2：STUN Client通过端口{IP-C1:Port-C1}向STUN Server{IP-S2:Port-S2}发送一个Binding Request（没有设置任何属性）。STUN Server收到该请求后，通过端口{IP-S2:Port-S2}把它所看到的STUN Client的IP和端口{IP-M1#2,Port-M1#2}作为Binding Response的内容回送给STUN Client。

**Test2：**STUN Client通过端口{IP-C1:Port-C1}向STUN Server{IP-S1:Port-S1}发送一个Binding Request（设置了Change IP和Change Port属性）。STUN Server收到该请求后，通过端口{IP-S2:Port-S2}把它所看到的STUN Client的IP和端口{IP-M2,Port-M2}作为Binding Response的内容回送给STUN Client。

**Test3：**STUN Client通过端口{IP-C1:Port-C1}向STUN Server{IP-S1:Port-S1}发送一个Binding Request（设置了Change Port属性）。STUN Server收到该请求后，通过端口{IP-S1:Port-S2}把它所看到的STUN Client的IP和端口{IP-M3,Port-M3}作为Binding Response的内容回送给STUN Client。

STUN协议的输出是：1）公网IP和Port2）防火墙是否设置3）客户端是否在NAT之后，及所处的NAT的类型

因此我们进而整理出，通过STUN协议，我们可以检测的类型一共有以下七种：

A：公开的互联网IP。主机拥有公网IP，并且没有防火墙，可自由与外部通信B：完全锥形NAT。C：受限制锥形NAT。D：端口受限制形NAT。E：对称型UDP防火墙。主机出口处没有NAT设备,但有防火墙,且防火墙规则如下：从主机UDP端口A发出的数据包保持源地址，但只有从之前该主机发出包的目的IP/PORT发出到该主机端口A的包才能通过防火墙。F：对称型NATG：防火墙限制UDP通信。

输入和输出准备好后，附上一张维基百科的流程图，就可以描述STUN协议的判断过程了。

![img](https://blog-10039692.file.myqcloud.com/1505808528013_2908_1505808528510.png)

**STEP1：检测客户端是否有能力进行UDP通信以及客户端是否位于NAT后 -- Test1**客户端建立UDP socket，然后用这个socket向服务器的（IP-1，Port-1）发送数据包要求服务器返回客户端的IP和Port，客户端发送请求后立即开始接受数据包。重复几次。a）如果每次都超时收不到服务器的响应，则说明客户端无法进行UDP通信，可能是：G防火墙阻止UDP通信b）如果能收到回应，则把服务器返回的客户端的（IP:PORT）同（Local IP: Local Port）比较：如果完全相同则客户端不在NAT后，这样的客户端是：A具有公网IP可以直接监听UDP端口接收数据进行通信或者E。否则客户端在NAT后要做进一步的NAT类型检测（继续）。

**STEP2：检测客户端防火墙类型 -- Test2**STUN客户端向STUN服务器发送请求，要求服务器从其他IP和PORT向客户端回复包：a）收不到服务器从其他IP地址的回复，认为包前被前置防火墙阻断，网络类型为Eb）收到则认为客户端处在一个开放的网络上，网络类型为A

**STEP3：检测客户端NAT是否是FULL CONE NAT -- Test2**客户端建立UDP socket然后用这个socket向服务器的(IP-1,Port-1)发送数据包要求服务器用另一对(IP-2,Port-2)响应客户端的请求往回发一个数据包，客户端发送请求后立即开始接受数据包。 重复这个过程若干次。a）如果每次都超时，无法接受到服务器的回应，则说明客户端的NAT不是一个Full Cone NAT，具体类型有待下一步检测（继续）。b）如果能够接受到服务器从(IP-2,Port-2)返回的应答UDP包，则说明客户端是一个Full Cone NAT，这样的客户端能够进行UDP-P2P通信。

**STEP4：检测客户端NAT是否是SYMMETRIC NAT -- Test1#2**客户端建立UDP socket然后用这个socket向服务器的(IP-1,Port-1)发送数据包要求服务器返回客户端的IP和Port, 客户端发送请求后立即开始接受数据包。 重复这个过程直到收到回应（一定能够收到，因为第一步保证了这个客户端可以进行UDP通信）。用同样的方法用一个socket向服务器的(IP-2,Port-2)发送数据包要求服务器返回客户端的IP和Port。比较上面两个过程从服务器返回的客户端(IP,Port),如果两个过程返回的(IP,Port)有一对不同则说明客户端为Symmetric NAT，这样的客户端无法进行UDP-P2P通信（检测停止）因为对称型NAT，每次连接端口都不一样，所以无法知道对称NAT的客户端，下一次会用什么端口。否则是Restricted Cone NAT，是否为Port Restricted Cone NAT有待检测（继续）。

**STEP5：检测客户端NAT是Restricted Cone 还是 Port Restricted Cone -- Test3**客户端建立UDP socket然后用这个socket向服务器的(IP-1,Port-1)发送数据包要求服务器用IP-1和一个不同于Port-1的端口发送一个UDP 数据包响应客户端, 客户端发送请求后立即开始接受数据包。重复这个过程若干次。如果每次都超时，无法接受到服务器的回应，则说明客户端是一个Port Restricted Cone NAT，如果能够收到服务器的响应则说明客户端是一个Restricted Cone NAT。以上两种NAT都可以进行UDP-P2P通信。

通过以上过程，至此，就可以分析和判断出客户端是否处于NAT之后，以及NAT的类型及其公网IP，以及判断客户端是否具备P2P通信的能力了

### ALG

### SBC

### ICE

### TURN

