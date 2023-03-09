# TCP 拥塞控制
一般来说，计算机网络都处在一个共享的环境。因此也有可能会因为其他主机之间的通信使得网络拥堵。
在网络出现拥堵时，如果继续发送大量数据包，可能会导致数据包时延、丢失等，这时 TCP 就会重传数据，但是一重传就会导致网络的负担更重，于是会导致更大的延迟以及更多的丢包，这个情况就会进入恶性循环被不断地放大
所以，TCP 不能忽略网络上发生的事，它被设计成一个无私的协议，当网络发送拥塞时，TCP 会自我牺牲，降低发送的数据量。
于是，就有了拥塞控制，控制的目的就是避免「发送方」的数据填满整个网络。
为了在「发送方」调节所要发送数据的量，定义了一个叫做「拥塞窗口」的概念。
拥塞窗口 cwnd 是发送方维护的一个的状态变量，它会**根据网络的拥塞程度动态变化**
拥塞窗口 cwnd 变化的规则：

- 只要网络中没有出现拥塞，cwnd 就会增大；
- 但网络中出现了拥塞，cwnd 就会减少；

只要「发送方」没有在规定时间内接收到 ACK 应答报文，也就是发生了**超时重传**，就会认为网络出现了拥塞。
拥塞控制主要是四个算法：**慢启动、拥塞避免、拥塞发生、快速恢复**
## 慢启动
TCP 在刚建立连接完成后，首先是有个慢启动的过程，这个慢启动的意思就是一点一点的提高发送数据包的数量，如果一上来就发大量的数据，这不是给网络添堵吗？
慢启动的算法记住一个规则就行：**当发送方每收到一个 ACK，拥塞窗口 cwnd 的大小就会加 1。**
慢启动算法的变化过程如下图：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1678336624029-5518443b-5de9-419c-9477-c2dc876ad9b3.png#averageHue=%23f9f9f8&clientId=u3c9c0346-5dc3-4&from=paste&height=411&id=u85376a26&name=image.png&originHeight=632&originWidth=1016&originalType=binary&ratio=2.5&rotation=0&showTitle=false&size=205052&status=done&style=none&taskId=u83cc60b4-7eec-4366-b80b-178c5079618&title=&width=661.4000244140625)
可以看出慢启动算法，发包的个数是**指数性增长**的
## 拥塞避免算法
当慢启动到一个阈值，也就是慢启动门限 ssthresh 

- 当 cwnd < ssthresh，使用慢启动算法
- 当 cwnd >= ssthresh，就会使用「拥塞避免算法」

那么进入拥塞避免算法后，它的规则是：**每当收到一个 ACK 时，cwnd 增加 1。（线性）**
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1678336845506-7eb876dc-1807-49b4-a12c-ef73dfc6a34c.png#averageHue=%23faf8f5&clientId=u3c9c0346-5dc3-4&from=paste&height=381&id=uc7e489ef&name=image.png&originHeight=731&originWidth=872&originalType=binary&ratio=2.5&rotation=0&showTitle=false&size=156333&status=done&style=none&taskId=u45a973b8-ec4c-4045-a897-255ebd782a4&title=&width=454.8000183105469)
## 拥塞发生算法
就这么一直增长着后，网络就会慢慢进入了拥塞的状况了，于是就会出现丢包现象，这时就需要对丢失的数据包进行重传。当触发了重传机制，也就进入了「拥塞发生算法」。
重传机制有两种：超时重传和快速重传
### 当发生超时重传时
这个时候，ssthresh 和 cwnd 的值会发生变化：

- ssthresh 设为 cwnd/2，
- cwnd 重置为 1 （是恢复为 cwnd **初始化值**，我这里假定 cwnd 初始化值 1）

![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1678337247589-7430157a-91b8-4070-a9e6-1cc2a405c7b8.png#averageHue=%23f8f5f2&clientId=u3c9c0346-5dc3-4&from=paste&height=627&id=ue95b7ef3&name=image.png&originHeight=873&originWidth=1142&originalType=binary&ratio=2.5&rotation=0&showTitle=false&size=307887&status=done&style=none&taskId=u088b7a70-4595-46b2-8890-098de24e50e&title=&width=819.7999877929688)
### 发生快速重传时
TCP 认为这种情况不严重，因为大部分没丢，只丢了一小部分，则 ssthresh 和 cwnd 变化如下：

- cwnd = cwnd/2 ，也就是设置为原来的一半;
- ssthresh = cwnd;

然后，进入快速恢复算法如下：

- 拥塞窗口 cwnd = ssthresh + 3 （ 3 的意思是确认有 3 个数据包被收到了）；
- 重传丢失的数据包；
- 如果再收到重复的 ACK，那么 cwnd 增加 1；
- 如果收到新数据的 ACK 后，把 cwnd 设置为第一步中的 ssthresh 的值，原因是该 ACK 确认了新的数据，说明从 duplicated ACK 时的数据都已收到，该恢复过程已经结束，可以回到恢复之前的状态了，也即再次进入拥塞避免状态；

![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1678337385465-7f0279ea-f534-4768-8e51-1e4d7a0f7abd.png#averageHue=%23f7f3ef&clientId=u3c9c0346-5dc3-4&from=paste&height=517&id=u82a27c9c&name=image.png&originHeight=873&originWidth=1352&originalType=binary&ratio=2.5&rotation=0&showTitle=false&size=408602&status=done&style=none&taskId=u83825c6f-9097-4ab5-b5ea-eb8d4fc848e&title=&width=800.7999877929688)
### 为什么在最后收到新的数据 ACK 之后，cwnd 设置回了 ssthresh？

1. 在快速恢复的过程中，首先 ssthresh = cwnd/2，然后 cwnd = ssthresh + 3，表示网络可能出现了阻塞，所以需要减小 cwnd 以避免，加 3 代表快速重传时已经确认接收到了 3 个重复的数据包；
2. 随后继续重传丢失的数据包，如果再收到重复的 ACK，那么 cwnd 增加 1。加 1 代表每个收到的重复的 ACK 包，都已经离开了网络。这个过程的目的是**尽快将丢失的数据包发给目标**。
3. 如果收到新数据的 ACK 后，把 cwnd 设置为第一步中的 ssthresh 的值，恢复过程结束。

首先，快速恢复是拥塞发生后慢启动的优化，其首要目的仍然是**降低 cwnd 来减缓拥塞**，所以必然会出现 cwnd 从大到小的改变。
其次，过程2（cwnd逐渐加1）的存在是为了**尽快将丢失的数据包发给目标**，从而解决拥塞的根本问题（三次相同的 ACK 导致的快速重传），所以这一过程中 cwnd 反而是逐渐增大的。
