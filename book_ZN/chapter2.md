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





