# ZeroNet 节点与追踪器\(Tracker\)通信分析

ZeroNet 节点与追踪器之间以通过HTTP、UDP 和Zero协议进行通信。其中HTTP和UDP遵循BitTorrent的BEP3标准，我们可以按照BitTorrent的标准通信方式进行分析\(有一个细节不同）。Zero协议用于访问onion结尾的服务器，这里不进行讨论。每一个ZeroNet网站都有一个和比特币收款地址格式相同格式的网站地址，如果ZeroNet要访问一个从未访问过的网站则会尝试向追踪器请求拥有此网站的节点列表。

## HTTP方式请求

HTTP方式是常用的请求方式，先以一个GET请求开始。\[BIP32\]\([**http://www.bittorrent.org/beps/bep\_0003.html**](http://www.bittorrent.org/beps/bep_0003.html "BEP3")\)

ZeroNet向跟踪器发出的请求如下：



