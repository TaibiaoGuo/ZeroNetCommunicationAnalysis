# ZeroNet节点与追踪器（Tracker）通信分析

ZeroNet节点与追踪器之间通过HTTP、UDP和Zero协议进行通信。其中HTTP和UDP遵循BitTorrent的BEP3标准，我们可以按照BitTorrent的标准通信方式进行分析（有一个细节不同）。Zero协议用于访问onion结尾的服务器，这里不进行讨论，每一个ZeroNet网站都有一个和比特币收款格式相同的网站地址，如果ZeroNet要访问一个从未访问过的网站则会尝试向追踪器请求拥有此网站的节点列表。

## HTTP请求方式

HTTP请求方式是最常用的请求方式，实现起来非常简单,见[BIP32](http://www.bittorrent.org/beps/bep_0003.html)。通信先从节点的GET请求开始，ZeroNet向跟踪器发出的请求如下：

```
[Full request URI: 
http://explodie.org:6969/announce?uploaded=0&downloaded=0&numwant=30&compact=1&event=started&peer_id=-ZN0060-mVNmLkbXhtty&port=15441&info_hash=%92g%25%D6%3Cb%86J%A3%DA%D2q%D6%CB%1B%A4%0E%0A%91%BF&left=0
]
```

`http://explodie.org`是跟踪器的网址，`6969`是通信端口，`announce`是一个习惯字段，说明这个网站提供跟踪器服务，`?`后面是GET请求，以&分隔请求键值对。

| 键 | 用途 |
| --- | --- |
| uploaded，downloaded | 节点告诉跟踪器自己目前的上传下载量 |
| numwant | 节点需要的拥有该网站的节点数 |
| compact | 是否以压缩方式返回节点列表 |
| event | 节点目前的状态，started，stop，completed |
| peer\_id | 节点的唯一编码，20位，由固定位和随机位构成 |
| port | 报告其他节点与自己连接的端口，若发送0则表明是内网IP |
| info\_hash | 网站的比特币地址用hash计算后的值再经过url编码得到info\_hash |
| left | 目前已经下载多少的偏移量 |

跟踪器返回一个由msgpack编码过后的节点表

`d12:min intervali1800e5:peers6:..:.<Q8:completei1058e10:incompletei0e8:intervali1800ee`

> MessagePack是一个基于二进制高效的对象序列化类库，可用于跨语言通信。它可以像JSON那样，在许多种语言之间交换结构对象；但是它比JSON更快速也更轻巧。支持Python、Ruby、Java、C/C++等众多语言。比Google Protocol Buffers还要快4倍。

在ZeroNet中作者大量用Msgpack替代json来进行数据传输，因为Msgpack是BitTorrent的bencode的超集，因此我认为ZeroNet作者大量使用Msgpack是为了减少改造BitTorrent的工作量。Msgpack是一种优秀的传输编码。但是就我看来，这种编码方式没有考虑多语种环境，在中文中使用的时候可能会引发崩溃，如果要使用我们需要先用Base64 先对中文做一次预处理然后在利用Msgpack进行编码，ZeroNet也是这样的思路。因此还是推荐使用Google Protocol Buffers 或者JSON。

Msgpack对于理解ZeroNet运行过程非常重要，本文附录有一个简单教程。[传送门]()

利用msgpack解码，我们得到了节点表的具体信息。

## UDP传输

使用HTTP虽然十分简单,但是使用HTTP会带来巨大的开销。如果使用UDP免去了建立和关闭连接的开销，大约可以减少50%左右的流量。此外，UDP的二进制协议不需要复杂的解析器和连接处理，降低了跟踪代码的复杂性并提高了性能。

因为UDP是一种“不可靠”的协议，这意味着他不会自己重传丢失的数据包。数据的完整性叫给上层的程序来负责。如果在15\*2^n秒后没有收到响应，则客户端重新发送请求，其中n从0开始，并在每次重新传输后增加到8。

**例子**

```
正常宣布：

t = 0：连接请求
t = 1：连接响应
t = 2：通知请求
t = 3：通知响应
连接超时：

t = 0：连接请求
t = 15：连接请求
t = 45：连接请求
t = 105：连接请求
等
宣布超时：

t = 0：
t = 0：连接请求
t = 1：连接响应
t = 2：通报请求
t = 17：通报请求
t = 47：通报请求
t = 107：连接请求（因连接ID过期）
t = 227：连接请求
等

多个请求：
t = 0: 请求连接
t = 1: 连接回应
t = 2: 宣布回应
t = 3: 宣布回应
t = 4: 宣布回应
t = 5：宣布回应
t = 60：宣布请求
t = 61：宣布回应
t = 62：连接请求
t = 63：连接响应
t = 64：宣布请求
t = 64：刮取请求
t = 64：刮取请求
t = 64：宣布请求
t = 65：宣布回应
t = 66：宣布回应
t = 67：刮响应
t = 68：刮刮响应


```

## UDP跟踪器协议

所有值都以网络字节顺序（大端）发送。不要期望数据包完全具有一定的大小。未来的扩展可能会增加数据包的大小。

###连接
在宣布或刮取之前，您必须获得连接ID。

1.  选择一个随机的交易ID。

2.  填充连接请求结构。
3.  发送数据包。

###连接请求

| 偏移大小 | 名称                                    |
| ---- | ------------------------------------- |
| 0    | 64位整数protocol_id 0x41727101980 //魔术常量 |
| 8    | 32位整数操作0 //连接                         |
| 12   | 32位整数transaction_id                   |
| 16   |                                       |



1.  接收数据包。

2.  检查数据包是否至少有16个字节。
3.  检查交易ID是否等于您选择的交易ID。
4.  检查操作是否连接。
5.  存储连接ID以供将来使用。

###连接响应

| 偏移大小 | 名称                  |
| ---- | ------------------- |
| 0    | 32位整数操作0 //连接       |
| 4    | 32位整数transaction_id |
| 8    | 64位整数connection_id  |
| 16   |                     |



###宣布

1.  选择一个随机的交易ID。
2.  填写通知请求结构。
3.  发送数据包。

**IPv4宣布请求:**

偏移大小名称值
0 64位整数connection_id
8 32位整数操作1 //宣布
12 32位整数transaction_id
16个20字节的字符串info_hash
36个20字节的字符串peer_id
56个64位整数下载
剩64个64位整数
72上传的64位整数
80 32位整数事件0 // 0：无; 1：完成; 2：开始; 3：停了
84 32位整数IP地址0 //默认
88 32位整数密钥
92 32位整数num_want -1 //默认值
96个16位整数端口
98
接收数据包。
检查数据包是否至少有20个字节。
检查交易ID是否等于您选择的交易ID。
检查行动是否宣布。
直到间隔秒数或事件发生之前，不要再次公布。
请注意，大多数追踪者只会在有限的情况下遵守IP地址字段。

IPv4宣布响应：

偏移大小名称值
0 32位整数行动1 //宣布
4 32位整数transaction_id
8位32位整数间隔
12个32位整数的leechers
16个32位整数播种机
20 + 6 * n 32位整数IP地址
24 + 6 * n 16位整数TCP端口
20 + 6 * N

## 如何对ZeroNet的代码进行更新
ZeroNet每次启动都会尝试连接

```
 urls = [
        "https://github.com/HelloZeroNet/ZeroNet/archive/master.zip",
        "https://gitlab.com/HelloZeroNet/ZeroNet/repository/archive.zip?ref=master",
        "https://try.gogs.io/ZeroNet/ZeroNet/archive/master.zip"
    ]
```

```
Update site path: data/1UPDatEDxnvHDo7TXvq6AEBARfNkyfxsp, bad_files: 0
Downloading from: https://github.com/HelloZeroNet/ZeroNet/archive/master.zip 
. . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . .
 . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 
 . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 
 . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . .
  . . . . . . . . . . . . Downloaded.
Plugins enabled: ['AnnounceZero', 'Bigfile', 'Cors', 'CryptMessage', 'FilePack', 
'MergerSite', 'Mute', 'Newsfeed', 'OptionalManager', 'PeerDb', 'Sidebar', 'Stats', 
'TranslateSite', 'Trayicon', 'Zeroname'] disabled: ['Bootstrapper', 'Dnschain',
 'DonationMessage', 'Multiuser', 'StemPort', 'UiPassword', 'Zeroname-local']
Extracting to D:\ZeroNet-master... . . . . . . . . . . . . . . P . P . P . P .
 P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P 
 . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . 
 P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P 
 . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . 
 P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P
  . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P 
  . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P 
  . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P . P 
  . P . P . P . P . P . P . P . P . P .P . P . P . P . P . P . P . P . P . P .
   P . P . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . .
    . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 
    . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . .
     . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 
     . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 
     . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 
     . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . .
      . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . .
       . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 
       . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 
       . . . . . . . . . Done.
     Press enter to exit
```

通过update模块进行源代码升级，因此，这个模块我们要改为自己的代码源或者注释掉此模块。
