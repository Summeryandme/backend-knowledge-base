# Redis 是单线程吗
Redis 单线程指的是「接收客户端请求 -> 解析请求 -> 进行数据读写等操作 -> 发送数据给客户端」这个过程是由主线程来完成的
但是，Redis 程序并不是单线程的，Redis 在启动时，会启动后台线程

- Redis 在 2.6 版本后，会启动 2 个后台线程，分别处理关闭文件、AOF 刷盘这两个任务
- Redis 在 4.0 版本后，新增了一个新的后台线程，用来异步释放 Redis 内存，也就是 lazyfree 线程。例如执行 unlink key / flushdb async / flushall async 等命令，会把这些删除操作交给后台线程来执行，好处是不会导致 Redis 主线程卡顿。因此，当我们要删除一个大 key 的时候，不要使用 del 命令删除，因为 del 是在主线程处理的，这样会导致 Redis 主线程卡顿，因此我们应该使用 unlink 命令来异步删除大 key。

之所以 Redis 为「关闭文件、AOF 刷盘、释放内存」这些任务创建单独的线程来处理，是因为这些任务的操作都是很耗时的，如果把这些任务都放在主线程来处理，那么 Redis 主线程就很容易发生阻塞，这样就无法处理后续的请求了
后台线程相当于一个消费者，生产者把耗时任务丢到任务队列中，消费者（BIO）不停轮询这个队列，拿出任务就去执行对应的方法即可
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1680074948302-e8b86678-c454-4c9e-8542-e68f639da066.png#averageHue=%23afaba8&clientId=udd9ab6d0-b328-4&from=paste&height=470&id=uf08b3dd2&name=image.png&originHeight=834&originWidth=1282&originalType=binary&ratio=2.5&rotation=0&showTitle=false&size=267556&status=done&style=none&taskId=ud97d9ca8-c9ee-4278-91bf-d383d278621&title=&width=721.7999877929688)
关闭文件、AOF 刷盘、释放内存这三个任务都有各自的任务队列：

- BIO_CLOSE_FILE，关闭文件任务队列：当队列有任务后，后台线程会调用 close(fd) ，将文件关闭；
- BIO_AOF_FSYNC，AOF 刷盘任务队列：当 AOF 日志配置成 everysec 选项后，主线程会把 AOF 写日志操作封装成一个任务，也放到队列中。当发现队列有任务后，后台线程会调用 fsync(fd)，将 AOF 文件刷盘，
- BIO_LAZY_FREE，lazy free 任务队列：当队列有任务后，后台线程会 free(obj) 释放对象 / free(dict) 删除数据库所有对象 / free(skiplist) 释放跳表对象；
# Redis 单线程模式
Redis 初始化的时候，会做下面几个事情

1. 调用 epoll_create() 创建一个 epoll 对象和调用 socket() 创建一个服务端 socket
2. 调用 bind() 绑定端口和调用 listen() 监听该 socket
3. 将调用 epoll_ctl() 将 listen socket 加入到 epoll，同时注册「连接事件」处理函数

初始化完成之后，主线程就进入到一个**事件循环函数**

- 首先，先调用处理发送队列函数，看是发送队列里是否有任务，如果有发送任务，则通过 write 函数将客户端发送缓存区里的数据发送出去，如果这一轮数据没有发送完，就会注册写事件处理函数，等待 epoll_wait 发现可写后再处理
- 接着，调用 epoll_wait 函数等待事件的到来
   - 如果是连接事件到来，则会调用连接事件处理函数，该函数会做这些事情：调用 accpet 获取已连接的 socket -> 调用 epoll_ctl 将已连接的 socket 加入到 epoll -> 注册「读事件」处理函数
   - 如果是读事件到来，则会调用读事件处理函数，该函数会做这些事情：调用 read 获取客户端发送的数据 -> 解析命令 -> 处理命令 -> 将客户端对象添加到发送队列 -> 将执行结果写到发送缓存区等待发送
   - 如果是写事件到来，则会调用写事件处理函数，该函数会做这些事情：通过 write 函数将客户端发送缓存区里的数据发送出去，如果这一轮数据没有发送完，就会继续注册写事件处理函数，等待 epoll_wait 发现可写后再处理
# 为何单线程这么快

- Redis 的大部分操作都在内存中进行，并且采用了高效的数据结构，因此 Redis 的性能瓶颈主要是内存和网络带宽，而并非 CPU，即然 CPU 并不是瓶颈，那么自然就会采用单线程的方案了
- Redis 采用单线程避免了多线程之间的竞争，省去了多线程切换带来的时间和性能上的开销，而且也没有死锁情况的发生
- Redis 采用了 I/O 多路复用机制处理大量的客户端 Socket 请求，IO 多路复用机制是指一个线程处理多个 IO 流，就是我们经常听到的 select/epoll 机制。简单来说，在 Redis 只运行单线程的情况下，该机制允许内核中，同时存在多个监听 Socket 和已连接 Socket。内核会一直监听这些 Socket 上的连接请求或数据请求。一旦有请求到达，就会交给 Redis 线程处理，这就实现了一个 Redis 线程处理多个 IO 流的效果
# Redis 6.0 之后为什么引入了多线程
虽然 Redis 的主要工作（网络 I/O 和执行命令）一直是单线程模型，但是在 Redis 6.0 版本之后，也采用了多个 I/O 线程来处理网络请求，这是因为随着网络硬件的性能提升，Redis 的性能瓶颈有时会出现在网络 I/O 的处理上。
所以为了提高网络 I/O 的并行度，Redis 6.0 对于网络 I/O 采用多线程来处理。但是对于命令的执行，Redis 仍然使用单线程来处理，所以不要误解 Redis 有多线程同时执行命令。
Redis 官方表示，Redis 6.0 版本引入的多线程 I/O 特性对性能提升至少是一倍以上。
Redis 6.0 版本支持的 I/O 多线程特性，默认情况下 I/O 多线程只针对发送响应数据（write client socket），并不会以多线程的方式处理读请求（read client socket）。要想开启多线程处理客户端读请求，就需要把 Redis.conf 配置文件中的 io-threads-do-reads 配置项设为 yes。
同时， Redis.conf 配置文件中提供了 IO 多线程个数的配置项，关于线程数的设置，官方的建议是如果为 4 核的 CPU，建议线程数设置为 2 或 3，如果为 8 核 CPU 建议线程数设置为 6，线程数一定要小于机器核数，线程数并不是越大越好。

