This file is licensed under the [MIT License (MIT)](http://opensource.org/licenses/MIT) available on http://opensource.org/licenses/MIT.

[对应英文文档](https://github.com/bitcoin-dot-org/bitcoin.org/blob/master/_includes/devdoc/guide_p2p_network.md)

# P2P网络
比特币网络协议允许全节点（peers）之间协作地维护一个区块和交易交换的p2p网络。全节点在把转发前先校验每个区块和交易。归档节点市值可以存储整个区块链的区块链并可以为其他节点提供历史区块的全节点。剪枝节点是不保存整个区块链的全节点。许多客户端通常使用比特币网络协议与全节点相连。

一致性协议并不包括网络校验，因此比特币程序可以调整网络和协议，比如一些矿工使用的高速区块交换网络、一些为钱包提供SPV级别安全的专门交易信息服务器等。

为了为比特币p2p网络提供一个使用的例子，本节当中使用比特币核心作为全节点的代表，BitcoinJ作为SPV客户端代表。这两个程序功能都是弹性的，因此这里只介绍基本功能。同时，出于隐私考虑，例子中出现的IP地址使用保留IP代替。

## 伙伴发现
第一次启动时，程序不知道任何的存活节点。为了发现一些IP地址，他们查询一个或多个在比特币核心和BitcoinJ程序中硬编码的DNS名字（称为DNS种子）。查询的结果将会包含一个或多个 DNS A记录，这些A记录会伴随一些可能接受新连接的全节点IP地址。比如，使用`dig`命令：
```
;; QUESTION SECTION:
;seed.bitcoin.sipa.be.	    IN  A

;; ANSWER SECTION:
seed.bitcoin.sipa.be.	60  IN  A  192.0.2.113
seed.bitcoin.sipa.be.	60  IN  A  198.51.100.231
seed.bitcoin.sipa.be.	60  IN  A  203.0.113.183
[...]
```

DNS种子由比特币社区的成员维护：一些人提供动通过扫描网络自动获取活跃IP地址的动态DNS种子服务器，一些人提供静态手动更新的可能不太活跃的IP的DNS种子。在一些情况下，在指定主网络端口8333或测试网络端口18333运行的节点会被自动加入DNS种子当中。

由于DNS种子不是权威认证的，一些恶意种子提供者或网络中间人攻击可能只返回被攻击者控制的种子，从而把程序孤立在攻击者自己的网络中，以此来发起伪造协议和区块。由于这些原因，程序不能完全只依靠DNS种子。

一旦一个程序接入网络中，它的伙伴就开始向它发送`addr`（地址）信息，其中个那些带了网络当中其他伙伴的端口和地址，提供了一种完全区中心的伙伴发现方法。比特币核心把当前已知的伙伴持久化保存在一个磁盘数据库当中，后面的启动中通常直接连接到这些伙伴而不再使用DNS种子。

但是，经常有伙伴离开网络或改变IP地址，因此程序可能启动前再一次成功的连接前发起多次尝试。这会为程序引入较大的延迟才能接入网络，迫使用户在发送交易或校验支付状态时等待很久。

为了避免这种可能的延迟，对于刚激活的节点，BitcoinJ通常使用动态DNS种子获取IP地址。比特币核心通常试图在最小化延迟和避免使用多余的DNS种子使用之间达到一个平衡：如果比特币核心在它的伙伴数据库当中有相应的记录，在使用种子前，它最多花费11s尝试连接至少一个伙伴；如果在这段时间内建立起一个连接，就不需要查询任何种子了。

比特币核心和BitcoinJ都包含一个硬编码的IP和端口列表，它们指向对应版本第一次发布时点附近活跃的节点。比特币核心在所有的DNS种子服务器在60s内没有响应时将开始尝试区连接到这些节点，提供一个自动的后备选项。

对于手动后备选项，比特币核心也提供了几个命令行连接选项，包括从通过IP指定的节点链接到一系列伙伴，或和某个通过IP指定的固定的节点维持一个持续连接。使用`-help`参数查看文本帮助信息。BitcoinJ也可以编程支持同样的操作。

资源：Bitcoin Seeder，该程序运行在比特币核心和BitcoinJ使用的种子服务器上。比特币核心DNS Seed Policy。比特币核心和BitcoinJ使用的硬编码的IP地址使使用makeseeds脚本产生的。

## 连接伙伴
连接是通过发送一个`version`信息到远端节点实现的，其中包括版本号、区块、当前时间。远端节点使用自己的版本信息回复。这样两个节点相互发送一个`verack`信息证实连接已经建立。

一旦建立连接，本地节点就可以向远端节点发送`getaddr`和`addr`信息获取更多的伙伴。

为了维护和伙伴的连接，节点默认在30分钟连接失效前向伙伴节点发送一条信息。如果90分钟内都没有收到一个伙伴的信息，节点就会假定伙伴的连接已经关闭。

## 初始区块下载
在一个全节点有能力校验一个未确认交易和最近新挖区块前，他必须下载并校验从第1个区块（硬编码的创始区块之后）开始到当前最优（可能存在分叉）区块链的所有区块。这被成为IBD（Initial Block Download，初始区块下载）或初始同步。

尽管“初始”这个词暗示这个过程只执行一次，但在任何时候有大量区块下载时候也都会使用它，比如之前接入的节点已经离线很久。这种情况下，节点可以使用IBD方式下载从它上次最后一次在线到现在的所有区块。

只要节点最优区块链上最新区块的区块头部时间距现在已经超过24小时时，比特币核心就会使用IBD方法。比特币核心0.10.0版本也会在本地最优区块链高度比本地最优头部链(header chain)的高度低144时（这也意味着本地区块链距离当前已经超过24小时）采用IBD。

### 区块优先
直到0.9.3版本，比特币核心都采用一种我们称之为区块优先的简单IBD（Initial Block Download，初始区块下载）方式。它的目标是从最新区块链序列中下载完整区块。

![image](http://note.youdao.com/yws/public/resource/580887bc0d4654b83202bc1e47db9af9/xmlnote/F25100822915449E8FDED35667ED9E49/118606)

当节点第一次启动时，它的本地区块链只包含一个节点——硬编码的创始区块（区块0）。这个节点选择一个远端伙伴，成为同步节点，然后向它发送一个如下所示的`getblocks`消息。

![image](http://note.youdao.com/yws/public/resource/580887bc0d4654b83202bc1e47db9af9/xmlnote/5336F486438942FB80DAFED1F4021392/118608)

在`getblocks`信息的header hashs字段，新节点发送了它拥有的唯一的创始区块的头部hash值（6f32...0000，内部字节序）。此外它还把结束hash字段全部置零，以请求最大的响应。

当收到`getblocks`消息后，同步节点拿到头部hash在它的区块链中搜索那个hash值对应的区块。它发现区块0匹配，因此回复从区块1开始的500个区块的来列表（500是针对`getblocks`消息回复的最大值）。它把这些列表放在如下所示的`inv`消息中。

![image](http://note.youdao.com/yws/public/resource/580887bc0d4654b83202bc1e47db9af9/xmlnote/76C4AF2B2D504D57AD6806E12094E759/118610)

列表项是网络上信息的唯一标识。每个列表包含一个对象实例的类型字段和唯一标识字段。对于区块，它的唯一标识是头部的hash值。

区块列表项出现在`inv`消息中的信息和它出现在区块链中的顺序一致，因此第一个`inv`消息中包含的表项是从区块1到区块501。（比如，上例中展示的区块1的头部hash为4860...0000）。

IBD节点使用受到的列表使用如下所示的`getdata`消息每次向同步节点请求128个区块。

![image](http://note.youdao.com/yws/public/resource/580887bc0d4654b83202bc1e47db9af9/xmlnote/B1F92F8ED00F4D51B47A915D8C219B43/118612)

对于区块优先的节点来说顺序请求和发送区块非常重要，因为当前区块的头部要参照前面区块的hash值。这就意味着IBD节点只有收到父区块，才能完整校验当前区块。由于父区块没有收到导致无法校验的区块成为孤儿区块，下面有一小节将详细讨论。

一收到`getdata`消息，同步节点就会返回请求的区块。每个区块都会使用串行区块格式通过`block`消息分别发送。`block`消息的格式如下所示。

![image](http://note.youdao.com/yws/public/resource/580887bc0d4654b83202bc1e47db9af9/xmlnote/0D09AD5BA7D545E7A55A2270F67C5F8B/118614)

IBD下载每个区块、校验后，在请求下一个之前没有请求过的区块，并维护一个最大为128的待下载队列。当它接收到已有列表当中的所有区块后，将发送向同步节点再发送一个请求最多500个区块的列表`getblocks`消息。第二个请求如下所示，其中包含多个头部hash值。

![image](http://note.youdao.com/yws/public/resource/580887bc0d4654b83202bc1e47db9af9/xmlnote/E082936AC9D940FFA5D104F329846E7C/118616)

一旦收到第二个`getblocks`消息，同步节点搜寻它的最优区块链，按照接收顺序依次逐个尝试，找到一个符合消息中头部hash的区块。如果找到一个匹配的，它将返回匹配点下一个区块开始的500个区块列表。如果除了停止hash外没有匹配的区块，它就假定两个节点相同的只有区块0，这是她就发送一个从区块1开始id`inv`消息（和前面多次讨论的`inv`消息相同）。

重复的搜索搜索过程，即便IBD节点本地区块链和同步节点本地区块链分叉的情况下，仍可以让同步节点发送有用的列表。这种分叉检查在IBD节点越接近区块链的顶端时越有效。

当IBD节点收到第二个`inv`消息，它将使用`getdata`消息请求这些区块，同步节点会使用`getblocks`消息响应，然后IBD节点会使用另外的`getblocks`消息请求更多的列表。这种循环抑制重复知道IBD节点同步到当前区块链顶端。在那个时候，节点会接收正常的区块广播（下面的章节讨论）发送的区块。

**区块优先的优点和缺点**

区块优先最大的优势是简单，最大的劣势是IBD节点依赖单个节点下载所有区块。它们会存在如下问题：
- 速度限制：所有的请求都发送到同步节点，如果同步节点有上传带宽限制，IBD节点的下载速度也会很慢。注意：如果一个同步节点离线，比特币核心将会继续从另外一个节点下载，但是它仍然只会一次从一个同步节点下载。
- 下载重启：同步节点可以向IBD节点发送非最优（但有效）的区块链。IBD节点必须等到IBD过程接近完成时才能判断自己在非最优链上，强制IBD节点必须通过另一个节点再次重启区块下载过程。比特币核心在不同的区块高度（开发者选定）设置了多个区块链检查点，可以帮助IBD节点检查到自己正在接收一个可选的区块链历史，从而让IBD节点在下载过程的早期就重启下载。
- 磁盘错误攻击：类似于下载重启，如果同步节点同步非最优（但有效）的区块链，并被存储在IBD节点的磁盘上，不仅浪费磁盘存储空间还可能将磁盘用垃圾数据填满。
- 占用大量内存：无论是恶意为之还是意外，同步节点都可能乱序发送区块，导致产生故而区块，要等父区块收到并校验。孤儿区块在等待校验器件必须存储在内存中，这可能产生大量的内存占用。

上述所有的问题已经被比特币核心0.10.0版本的块头优先IBD方法部分或全部地解决了。

资源：下面的表格总结了本节提到的信息。信息的连接可以带你查看对应信息的参考页面。

| Message   | [getblocks]   | [inv]         | [getdata]     | [block]       |
| ---       | ---           | ---           | ---           | ---           |
| From > To | IBD -> Sync   | Sync -> IBD   | IBD -> Sync   | Sync -> IBD   |
| 内容      | 一个或多个头部hash     | 最多500个区块列表（唯一标识） |  一个或多个区块列表  | 一个顺序的区块    |

[getblocks]:(https://bitcoin.org/en/developer-reference#getblocks)
[inv]:(https://bitcoin.org/en/developer-reference#inv)
[getdata]:(https://bitcoin.org/en/developer-reference#getdata)
[block]:(https://bitcoin.org/en/developer-reference#block)

### 块头优先

比特币核心0.10.0使用了一种称作`块头优先`的IBD方法。目标是从最优块头链中下载块头，一边尽可能的校验，同时并行的下载对应的区块。这种方式解决了老的区块优先的IBD方法的许多问题。

![image](http://note.youdao.com/yws/public/resource/580887bc0d4654b83202bc1e47db9af9/xmlnote/A4079EF1FE0540CC81E766DAD198E98B/118618)

当节点第一次启动时，它的本地区块链只包含一个节点——硬编码的创始区块（区块0）。这个节点选择一个远端伙伴，成为同步节点，然后向它发送一个如下所示的`getheaders`消息。

![image](http://note.youdao.com/yws/public/resource/580887bc0d4654b83202bc1e47db9af9/xmlnote/0AD0DC8F1D27412A883084A0DAF3F3EB/118621)

在`getheaders`信息的header hashs字段，新节点发送了它仅有的创世区块的头部hash值（6f32...0000，内部字节序）。此外它还把结束hash字段全部置零，以请求最大的响应。

当收到`getheaders`消息后，同步节点拿到头部hash在它的区块链中搜索那个hash值对应的区块。它发现区块0匹配，因此回复从区块1开始的2000个头部（回复的最大值）。它把这些头部hash放在如下所示的`headers`消息中。

![image](http://note.youdao.com/yws/public/resource/580887bc0d4654b83202bc1e47db9af9/xmlnote/0AD0DC8F1D27412A883084A0DAF3F3EB/118621)

IBD节点暂时只能部分校验区块头，包括确保所有的字段符合一致性规则，以及根据nBits属性头部hash在目标门限以下。（完整校验仍然需要从对应区块中找回所有交易）

当IBD节点部分验证区块头部以后，它可以并行进行以下两项工作：
1. 下载更多的头部。IBD节点可以发送另一个`getheaders`消息到同步节点，请求最优块头链上下一组2000区块头部。这些块头可以立即被校验，然后再发出下一批请求如此重复，直到从同步节点收到的头部个数小于2000，意味着同步节点已经不能提供更多的头部。这段介绍写下的时间，块头同步可以在200次循环过程内完成，大约下载32MB的数据。
    一旦IBD节点从同步节点收到一个少于2000个块头的`headers`消息，它向周围的每个outbound 伙伴发送一个`getheaders`消息来获取他们的最优头部链信息。通过对比每个outbound伙伴的响应，它很容易就可以判定自己下载的块头是不是属于最优块头链。这就意味着即使不适用检查点，不诚实的同步节点也会很快被发现（IBD节点至少要连接一个诚实节点；比特币核心仍会继续提供检查点，以防一个诚实节点都找不到）。
2. 下载区块。当IBD节点不断下载块头以及块头下载完成时，它还在逐个请求并下载区块。IBD节点可以使用从头部链中计算得到的toubuhash创建一个`getdata`消息，通过他们的列表请求区块。它并一定非要从同步节点下载区块，任何一个全节点伙伴都可以。（虽然并不是所有的全节点都保存了全部区块）这样可以并行的获取区块，而且避免了下载速度受单个同步节点上行速度限制的问题。
    为了使用多个伙伴分担负载，比特币核心每次向每个伙伴最多可以请求16个区块。考虑到一个节点最多可以有8个outboud连接，块头优先的比特币核心在IBD期间可以最多可以同时请求128个区块（和区块优先比特币核心允许一次向同步节点请求的最大值相同）

![image](http://note.youdao.com/yws/public/resource/580887bc0d4654b83202bc1e47db9af9/xmlnote/EC75935230A84739A94D2D0FDC01B6AC/118623)

比特币核心块头优先模式使用一个1024区块的滑动下载窗口最大化下载速度。窗口的当前最小高度区块是下一个将被校验的，如果比特币核心准备好校验它时它仍为到达，比特币核心将会等待最少2s的时间防止发送延迟。如果区块仍未到达，比特币核心将断开和拖延节点的连接并尝试连接另外的节点。例如，如上所示，如果在2s内仍然没有发送区块3，那么和它的连接就会被断开。
一旦IBD节点同步到区块链当前最新位置，它就开始接收下一节介绍的正常的广播区块。
    
资源：下面的表格总结了本节提到的信息。信息的连接可以带你查看对应信息的参考页面。

| Message   | [getheaders]   | [headers]         | [getdata]     | [block]       |
| ---       | ---           | ---           | ---           | ---           |
| From > To | IBD -> Sync   | Sync -> IBD   | IBD -> Sync   | Sync -> IBD   |
| 内容      | 一个或多个头部hash     | 最多2000个区块头部 |  一个或多个区块列表  | 一个顺序的区块    |

[getheaders]:(https://bitcoin.org/en/developer-reference#getheaders)
[headers]:(https://bitcoin.org/en/developer-reference#headers)
[getdata]:(https://bitcoin.org/en/developer-reference#getdata)
[block]:(https://bitcoin.org/en/developer-reference#block)

## 区块广播
当一个矿工挖到一个新的区块，它将使用如下方式的一种将该区块广播到自己的伙伴节点：
- 自主区块推送：矿工把新区块放到一个`block`消息中发送给自己的全节点同伴。通过这种方式矿工可以合理的规避标准的同步方法，他知道自己所有的同伴都没有这个新发现的区块。
- 标准区块中继：矿工标准的中继节点一样，向他所有的伙伴节点（包括全节点和SPV节点）发送一个包含新区块列表信息的`inv`消息。通常回应有如下集中：
    - 每个块头优先同伴通过`getheaders`消息求区块，请求当中包含它的最优块头连中高度最高的块头的hash值，同样地请求当中会包含最优块头链中之前的区块头以便进行分叉监测。跟随这个消息后还会携带一个`getdata`消息请求完整区块。通过先请求区块，一个块头优先节点可以拒绝下面章节介绍的孤儿区块。
    - 每个SPV客户端通过发送一个`getdata`消息，通常请求merkle区块。

    矿工根据请求回复给每个节点，有的回复一个包含区块的`block`消息，或者一个或多个`headers`消息，或者merkle区块以及和包含SPV客户端布隆滤波器相关交易的`merkleblock`消息（消息后含0个或多个`tx`消息）。


默认情况下，比特币核心使用标准的区块中继方式广播区块，但是它也接受使用上面介绍的任何一种方式的广播的区块。

全节点校验接收到的区块，然后建议它的伙伴节点使用上面讨论的标准区块中继方法。下面浓缩的表格当中高亮了上面讨论的操作使用的消息（Relay, BF, HF 和SPV 代指中继节点、区块优先节点、块头优先节点和SPV客户端，any代指使用任意区块请求方法的节点）。

| Message   | [inv]     | [getdata] | [getheaders]  | [headers] |
| --        | ---       | ---       | ---           | ---       |
| From -> To|Realay->Any| BF->Relay |HF->Relay      | Relay->HF
| 负载      |新区块列表 |新区块列表 |HF节点最优头部链一个或多个头部hash | 中继链上与HF节点最优头部链相关的最多2000个头部

| Message   | [block]       | [merkleblock]         | [tx]  |  
| --        | ---           | ---                   | ---   |
|From -> To |Relay->BF/HF   |Relay->SPV             | Relay->SPV
| 负载      |顺序的新区块   |merkle区块过滤的新区块 | 新区块中满足布隆滤波器的顺序交易

[merkleblock]:https://bitcoin.org/en/developer-reference#merkleblock
[tx]:https://bitcoin.org/en/developer-reference#tx


### 孤儿区块
区块优先的节点可能会下载孤儿区块（把头部hash字段在当前区块头部的前一个节点尚未到达，本区块称为孤儿区块）。换句话说，孤儿区块没有已知的父区块（不同于废弃区块，废弃区块父区块已知，但是不属于最优区块链）。

![image](http://note.youdao.com/yws/public/resource/580887bc0d4654b83202bc1e47db9af9/xmlnote/58DEC8AEAC404D4C877F7D91CD30F408/119218)

块头优先节下载点下载到一个孤儿区块后并不对它进行校验。而是向发送孤儿区块的广播节点发送一个`getblock`消息，该广播节点会回复一个包含下载节点丢失区块列表（最多500条）的`inv`消息，下载节点将会使用一个`getdata`消息请求这些区块，广播节点将会通过`block`消息发送这些区块。下载节点将会校验这些区块，一旦前面孤儿区块的父区块被校验，它就会校验该孤儿区块。

块头优先节点通常先使用`getheaders`消息请求块头然后再用`getdata`消息请求区块的方式来避免这个复杂的过程。广播节点将会将会下发一个包含它认为下载节点更新到最新需要的所有（最多2000）区块块头的`headers`消息，这些头部将会指向他们的父节点，因此下载节点收到`blocks`消息中包含的区块肯定不是父区块，因为它所有的符节点都是已知的（即使尚未校验）。除此之外，如果收到的`block`消息中包含了一个孤儿区块，块头优先节点将会立刻忽略该区块。

但是忽略孤儿区块意味着头部优先节点将会忽略矿工通过自助区块推送发送的孤儿区块。

## 交易广播
为了把交易发送给同伴，需要使用`inv`消息。如果收到一个`getdata`消息的回复，就使用`tx`消息发送交易。同伴接收到交易后，如果交易有效，也使用同样的方式转发。

### 内存池
全信息节点会跟踪有资格被包含到下一个区块的未证实的交易。这对于实际挖掘这些区块的矿工来说更是必要的，但是对于想要追踪那些未确认交易的伙伴节点来说也是有用的，比如为SPV节点提供为确认交易信息的节点。

由于为确认的交易在比特币中没有永久的状态，比特币核心把他们存储在非持久内存中，称为内存池(memory pool)或存储池(mempool)。当一个节点关闭后，除了存储在它钱包中的交易外，存储池中的内容就会丢失。这意味着那些从没有被矿工确认的交易，随着节点重启或内存淘汰慢慢就会消失在网络中。

**那些被挖到废弃区块中的交易可能被重新加入内存池**。这些重新添加的交易也可能由于被包含在新的替换区块（废弃区块重组后添加到当前最优链）中而被立即删除。比特币核心现在的情况时，从废弃链顶端开始，一个一个移除废弃链上的区块。随着每个区块被删除，其中的交易会被添加回内存池。等到所有的废弃区块都被溢出，更新的区块会一个一个被添加到区块上，知道最先的区块。随着区块的添加，它包含的交易会被从内存池中删除。

SPV节点没有内存池，因此他们不中继交易。他们不能独立的校验一个没有被包含在区块中的交易是否被双花，因此他们不知道哪个区块有资格被加入到下一个区块。

## 异常节点
两种类型的广播都要注意，对于那些通过发送错误信息占用带宽和计算资源的异常节点室友惩罚机制的。如果一个节点的惩罚因子大于`-banscore=<n>`门限值，就会被禁止`-bantime=<n>`指定的一段时间（秒为单位，默认是86400，即24小时）。

## 警告
从比特币核心v0.13.0版本已经移除。

早期版本的比特币核心孕育开发者和受信任的团体成员提交比特币警告事件通知用户严重的大范围网络问题。这个消息系统从比特币核心v0.13.0就退休了。但是内部的警告、部分检查警告和`-alertnotify`选项依然保留。







