# HTTP 和 HTTPS 有什么区别
- HTTP 是超文本传输协议，信息是明文传输，存在安全风险等问题，HTTPS 则解决 HTTP 不安全的问题，在 TCP 和 HTTP 之间加了一层 SSL/TLS 安全协议，使得报文可以加密传输
- HTTP 连接建立相对简单，TCP 三次握手之后便可以进行 HTTP 的报文传输，而 HTTPS 在 TCP 三次握手之后，还需要进行 SSL/TLS 的握手过程，才可以进入加密报文传输
- 两者默认的端口号不同，HTTP 默认端口号是 80，HTTPS 默认端口号是 443
- HTTPS 协议需要向 CA（证书颁发机构）申请数字证书，来保证服务器的身份是可信的
# HTTPS 解决了哪些问题
HTTP 由于是明文传输，存在窃听、篡改、冒充等风险，而 HTTPS 可以很好解决这些问题

- 信息加密：交互信息无法被窃取
   - 使用混合加密的方式实现
- 校验机制：无法篡改通信内容，篡改了就无法显示
   - 使用摘要算法来实现完整性，能够生成独一无二的指纹
- 身份证书：可以验证服务器的身份
   - 将服务器的公钥放入到数字证书中，解决了冒充的风险
## 混合加密
HTTPS 采用的是对称加密和非对称加密结合的「混合加密」的方式

- 在通信建立之前采用非对称加密的方式交换「会话密钥」，后续就不再使用非对称加密
- 在通信过程中使用对称加密的「会话密钥」来加密明文数据
## 摘要算法 + 数字签名
为了保证传输的内容不被篡改，我们需要对内容计算一个「指纹」，然后和内容一起传送给对方，对方收到之后，先是对内容也计算出一个「指纹」，然后跟发送方发送的「指纹」做一个比较，如果「指纹」相同，说明内容没有被篡改，否则就可以判断出内容被篡改，在计算机中会使用摘要算法（哈希函数）来计算内容的哈希值，也就是「指纹」，这个哈希值是唯一的，且无法通过哈希值推导出内容

![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1680436455530-842682ba-d174-47ab-a673-a64c133e86e5.png#averageHue=%23f7f4ea&clientId=u6ed1d36a-a94e-4&from=paste&height=311&id=u75f27306&name=image.png&originHeight=636&originWidth=1276&originalType=binary&ratio=3&rotation=0&showTitle=false&size=209359&status=done&style=none&taskId=u899d797d-9057-4c79-86ce-c971e5a2ac1&title=&width=624.3333740234375)

通过哈希算法可以确保消息没有被篡改，但是并不能保证「内容 + 哈希值」不会被中间人替换，因为这里**缺少对客户端收到的消息是否来源于服务端的证明**
## 数字证书
前面可以知道

- 可以通过哈希算法来保证消息的完整性
- 可以通过数字签名来保证消息的来源可靠性（能确定消息是由持有私钥的一方发送的）

可是这还远远不够，还缺少身份验证的环节，**万一公钥是伪造的呢**？

这时候就需要一个权威的机构 CA（数字证书认证机构），将服务器公钥放在数字证书（由数字证书认证机构颁发）中，只要证书是可信的，公钥就是可信的

![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1680439549309-2b2996ee-fa0b-49e5-b4a6-4eeac0a39676.png#averageHue=%23dfd49c&clientId=u6ed1d36a-a94e-4&from=paste&height=486&id=u06e5abd1&name=image.png&originHeight=577&originWidth=779&originalType=binary&ratio=3&rotation=0&showTitle=false&size=318092&status=done&style=none&taskId=u2b578829-015c-4d59-a472-4498ada9456&title=&width=656.6666870117188)

# HTTP/1.1、HTTP/2、HTTP/3 演变
## HTTP/1.1 相比 HTTP/1.0 提高了什么性能
HTTP/1.1 相比 HTTP/1.0 性能上的改进：

- 使用长连接的方式改善了 HTTP/1.0 短连接造成的性能开销
- 支持管道（pipeline）网络传输，只要第一个请求发出去了，不必等其回来，就可以发第二个请求出去，可以减少整体的响应时间

但 HTTP/1.1 还是有性能瓶颈：

- 请求 / 响应头部（Header）未经压缩就发送，首部信息越多延迟越大。只能压缩 Body 的部分
- 发送冗长的首部。每次互相发送相同的首部造成的浪费较多
- 服务器是按请求的顺序响应的，如果服务器响应慢，会招致客户端一直请求不到数据，也就是队头阻塞
- 没有请求优先级控制
- 请求只能从客户端开始，服务器只能被动响应
## HTTP/2 做了什么优化
HTTP/2 协议是基于 HTTPS 的，所以 HTTP/2 的安全性也是有保障的

![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1680440101196-e0a5c67d-4fa3-4909-81dc-9e6e5d670f9a.png#averageHue=%23d5e1cb&clientId=u6ed1d36a-a94e-4&from=paste&height=383&id=u6a6cda86&name=image.png&originHeight=366&originWidth=587&originalType=binary&ratio=3&rotation=0&showTitle=false&size=138389&status=done&style=none&taskId=u19ca789f-67b9-4029-949f-d4937934acc&title=&width=613.6666717529297)

那 HTTP/2 相比 HTTP/1.1 性能上的改进：

- 头部压缩
- 二进制格式
- 并发传输
- 服务器主动推进资源
### 头部压缩
HTTP/2 会压缩头（Header）如果你同时发出多个请求，他们的头是一样的或是相似的，那么，协议会帮你消除重复的部分，这就是所谓的`HPACK`算法：在客户端和服务器同时维护一张头信息表，所有字段都会存入这个表，生成一个索引号，以后就不发送同样字段了，只发送索引号，这样就提高速度了。
### 二进制格式
HTTP/2 不再像 HTTP/1.1 里的纯文本形式的报文，而是全面采用了二进制格式，头信息和数据体都是二进制，并且统称为帧（frame）：头信息帧（Headers Frame）和数据帧（Data Frame）

![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1680440247057-587cd5fe-0d27-4a8e-b8f3-03998bbbdab7.png#averageHue=%23e2ded8&clientId=u6ed1d36a-a94e-4&from=paste&height=319&id=uc2ec3f08&name=image.png&originHeight=564&originWidth=1111&originalType=binary&ratio=3&rotation=0&showTitle=false&size=169285&status=done&style=none&taskId=ud8bc2f89-e353-435c-b52c-e817cec2558&title=&width=629.3333740234375)

这样虽然对人不友好，但是对计算机非常友好，因为计算机只懂二进制，那么收到报文后，无需再将明文的报文转成二进制，而是直接解析二进制报文，这增加了数据传输的效率

例如状态码 200，在 HTTP/1.1 中是用 '2', '0', '0' 三个字符来表示的（二进制：00110010 00110000 00110000），共用了 3 个字节，在 HTTP/2 对于状态码 200 的二进制编码是 10001000，只用了 1 字节就能表示，相比于 HTTP/1.1 节省了 2 个字节，对于 10001000

- 最前面的 1 标识该 Header 是静态表中已经存在的 KV
- 在静态表里，“:status: 200 ok” 静态表编码是 8，二进制即是 1000

因此，整体加起来就是 1000 1000

### 并发传输
我们都知道 HTTP/1.1 的实现是基于请求-响应模型的。同一个连接中，HTTP 完成一个事务（请求与响应），才能处理下一个事务，也就是说在发出请求等待响应的过程中，是没办法做其他事情的，如果响应迟迟不来，那么后续的请求是无法发送的，也造成了队头阻塞的问题

而 HTTP/2 引出了 Stream 概念，多个 Stream 复用在一条 TCP 连接，1 个 TCP 连接包含多个 Stream，Stream 里可以包含 1 个或多个 Message，Message 对应 HTTP/1 中的请求或响应，由 HTTP 头部和包体构成。Message 里包含一条或者多个 Frame，Frame 是 HTTP/2 最小单位，以二进制压缩格式存放 HTTP/1 中的内容（头部和包体）。

针对不同的 HTTP 请求用独一无二的 Stream ID 来区分，接收端可以通过 Stream ID 有序组装成 HTTP 消息，不同 Stream 的帧是可以乱序发送的，因此可以并发不同的 Stream ，也就是 HTTP/2 可以并行交错地发送请求和响应。

比如下图，服务端并行交错地发送了两个响应： Stream 1 和 Stream 3，这两个 Stream 都是跑在一个 TCP 连接上，客户端收到后，会根据相同的 Stream ID 有序组装成 HTTP 消息

![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1680440703590-00a7da88-9446-4452-99a4-6f37320910f5.png#averageHue=%23e8e6e4&clientId=u6ed1d36a-a94e-4&from=paste&height=192&id=u847a80a0&name=image.png&originHeight=228&originWidth=787&originalType=binary&ratio=3&rotation=0&showTitle=false&size=87080&status=done&style=none&taskId=u5ba1e0d8-f449-4763-8d4a-df0ef9ec8dd&title=&width=662.3333740234375)

### 服务器推送
HTTP/2 还在一定程度上改善了传统的「请求 - 应答」工作模式，服务端不再是被动地响应，可以主动向客户端发送消息，客户端和服务器双方都可以建立 Stream， Stream ID 也是有区别的，**客户端建立的 Stream 必须是奇数号，而服务器建立的 Stream 必须是偶数号**

比如下图，Stream 1 是客户端向服务端请求的资源，属于客户端建立的 Stream，所以该 Stream 的 ID 是奇数（数字 1）；Stream 2 和 4 都是服务端主动向客户端推送的资源，属于服务端建立的 Stream，所以这两个 Stream 的 ID 是偶数（数字 2 和 4）。

![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1680440791180-fd8832bf-4f5b-450e-91e5-dcbd909d5c81.png#averageHue=%23ebeae9&clientId=u6ed1d36a-a94e-4&from=paste&height=282&id=ua353883d&name=image.png&originHeight=598&originWidth=1482&originalType=binary&ratio=3&rotation=0&showTitle=false&size=242713&status=done&style=none&taskId=u24953224-73df-4b03-906c-8057242547a&title=&width=700)

再比如，客户端通过 HTTP/1.1 请求从服务器那获取到了 HTML 文件，而 HTML 可能还需要依赖 CSS 来渲染页面，这时客户端还要再发起获取 CSS 文件的请求，需要两次消息往返，如下图左边部分

![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1680440912526-bfb69a4d-eaaf-41e6-adfa-a3d5f2307639.png#averageHue=%23f7f7f7&clientId=u6ed1d36a-a94e-4&from=paste&height=349&id=u09f5024c&name=image.png&originHeight=402&originWidth=800&originalType=binary&ratio=3&rotation=0&showTitle=false&size=53101&status=done&style=none&taskId=uec044aaf-2014-405d-8fdf-e0ec80bd103&title=&width=694.6666870117188)

如上图右边部分，在 HTTP/2 中，客户端在访问 HTML 时，服务器可以直接主动推送 CSS 文件，减少了消息传递的次数

## HTTP/2 有什么缺陷
HTTP/2 通过 Stream 的并发能力，解决了 HTTP/1 队头阻塞的问题，看似很完美了，但是 HTTP/2 还是存在“队头阻塞”的问题，只不过问题不是在 HTTP 这一层面，而是在 TCP 这一层，HTTP/2 是基于 TCP 协议来传输数据的，TCP 是字节流协议，TCP 层必须保证收到的字节数据是完整且连续的，这样内核才会将缓冲区里的数据返回给 HTTP 应用，那么当「前 1 个字节数据」没有到达时，后收到的字节数据只能存放在内核缓冲区里，只有等到这 1 个字节数据到达时，HTTP/2 应用层才能从内核中拿到数据，这就是 HTTP/2 队头阻塞问题
## HTTP/3 做了哪些优化
前面可以得知 HTTP/1.1 和 HTTP/2 都有队头阻塞的问题

- HTTP/1.1 中的管道（pipeline）虽然解决了请求的队头阻塞，但是没有解决响应的队头阻塞，因为服务端需要按顺序响应收到的请求，如果服务端处理某个请求消耗的时间比较长，那么只能等响应完这个请求后， 才能处理下一个请求，这属于 HTTP 层队头阻塞
- HTTP/2 虽然通过多个请求复用一个 TCP 连接解决了 HTTP 的队头阻塞 ，但是一旦发生丢包，就会阻塞住所有的 HTTP 请求，这属于 TCP 层队头阻塞

HTTP/2 队头阻塞的问题是因为 TCP，所以 HTTP/3 把 HTTP 下层的 TCP 协议改成了 UDP！

![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1680441074516-a00aecc3-4e7b-4385-9e12-46918bf90d18.png#averageHue=%23d7ce5a&clientId=u6ed1d36a-a94e-4&from=paste&height=333&id=u166229b4&name=image.png&originHeight=366&originWidth=782&originalType=binary&ratio=3&rotation=0&showTitle=false&size=154875&status=done&style=none&taskId=u26ebdc31-f603-4afb-9a19-a616e0031fb&title=&width=711.6666870117188)

UDP 发送是不管顺序，也不管丢包的，所以不会出现像 HTTP/2 队头阻塞的问题。大家都知道 UDP 是不可靠传输的，但基于 UDP 的 QUIC 协议 可以实现类似 TCP 的可靠性传输，QUIC 有以下 3 个特点

- 无队头阻塞
- 更快的连接建立
- 连接迁移
### 无队头阻塞
QUIC 协议也有类似 HTTP/2 Stream 与多路复用的概念，也是可以在同一条连接上并发传输多个 Stream，Stream 可以认为就是一条 HTTP 请求，QUIC 有自己的一套机制可以保证传输的可靠性的。当某个流发生丢包时，只会阻塞这个流，其他流不会受到影响，因此不存在队头阻塞问题。这与 HTTP/2 不同，HTTP/2 只要某个流中的数据包丢失了，其他流也会因此受影响
### 更快的连接建立
对于 HTTP/1 和 HTTP/2 协议，TCP 和 TLS 是分层的，分别属于内核实现的传输层、openssl 库实现的表示层，因此它们难以合并在一起，需要分批次来握手，先 TCP 握手，再 TLS 握手。HTTP/3 在传输数据前虽然需要 QUIC 协议握手，但是这个握手过程只需要 1 RTT，握手的目的是为确认双方的「连接 ID」，连接迁移就是基于连接 ID 实现的。但是 HTTP/3 的 QUIC 协议并不是与 TLS 分层，而是 QUIC 内部包含了 TLS，它在自己的帧会携带 TLS 里的“记录”，再加上 QUIC 使用的是 TLS/1.3，因此仅需 1 个 RTT 就可以「同时」完成建立连接与密钥协商，甚至，在第二次连接的时候，应用数据包可以和 QUIC 握手信息（连接信息 + TLS 信息）一起发送，达到 0-RTT 的效果
### 连接迁移
基于 TCP 传输协议的 HTTP 协议，由于是通过四元组（源 IP、源端口、目的 IP、目的端口）确定一条 TCP 连接，那么当移动设备的网络从 4G 切换到 WIFI 时，意味着 IP 地址变化了，那么就必须要断开连接，然后重新建立连接。而建立连接的过程包含 TCP 三次握手和 TLS 四次握手的时延，以及 TCP 慢启动的减速过程，给用户的感觉就是网络突然卡顿了一下，因此连接的迁移成本是很高的

而 QUIC 协议没有用四元组的方式来“绑定”连接，而是通过连接 ID 来标记通信的两个端点，客户端和服务器可以各自选择一组 ID 来标记自己，因此即使移动设备的网络变化后，导致 IP 地址变化了，只要仍保有上下文信息（比如连接 ID、TLS 密钥等），就可以“无缝”地复用原连接，消除重连的成本，没有丝毫卡顿感，达到了连接迁移的功能

所以， QUIC 是一个在 UDP 之上的伪 TCP + TLS + HTTP/2 的多路复用的协议。但 QUIC 是新协议，对于很多网络设备，根本不知道什么是 QUIC，只会当做 UDP，这样会出现新的问题，因为有的网络设备是会丢掉 UDP 包的，而 QUIC 是基于 UDP 实现的，那么如果网络设备无法识别这个是 QUIC 包，那么就会当作 UDP包，然后被丢弃。















