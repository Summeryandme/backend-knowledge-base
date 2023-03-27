# 概述
- 内存快照（redis data base）RDS
   - 将当前数据保存到硬盘
- 增量日志（append only file）AOF
   - 将每次执行的写命令保存到硬盘，类似于 mysql 的 Binlog
# RDB
指定的时间间隔内将内存中的数据库快照写入磁盘，是以内存数据的二进制序列化的形式进行的持久化，每次都要从 Redis 中生成一个快照进行数据的全量备份
优点：

- 存储紧凑，节省内存空间
- 恢复速度很快
- 适合全量备份，全量复制的场景，经常用于灾难恢复

缺点：

- 容易丢失数据，容易丢失两次快照之间 Redis 服务器中变化的数据
- RDB 通过 fork 子线程对内存快照进行全量备份，是一个重量级的操作，频繁执行成本较高
### dump.rdb 文件结构

- 长度为 5 字节的 REDIS 常量字符串
- 四字节的 db_version，标识 RDB 文件版本
- databases：不定长度，包含零个或者多个数据库，以及各数据库之间的键值对数据
- 1字节的EOF常量，表示文件正文内容结束
- check_sum：8 字节长的无符号整数，保存校验和
### RDB 文件的创建
#### 手动指令触发

- save：阻塞 Redis 的其他操作，会导致 Redis 无法响应客户端请求，不建议使用
- bgsave：Redis后台创建子线程，异步进行快照的保存操作，此时 Redis 仍然能响应客户端的请求
#### 自动间隔性保存
**_“N 秒内数据集至少有 M 个改动”_**
save 60 10 -> 60秒内有至少有10个键被改动
Redis默认配置如下
save 60 10000
save 300 10
save 900 1
#### 自动保存配置的数据结构
记录了服务器触发自动`bgsave`的`saveparams`属性

- `lastsave`：记录服务器最后一次执行`SAVE`或者`BGSAEVE`的时间
- `dirty`：自最后一次保存 RDB 文件以来，服务器进行了多少次写入
### 备份过程
![image.png](https://cdn.nlark.com/yuque/0/2022/png/28316065/1666939540280-6744a284-1993-4592-a56d-a66a23f9c906.png#averageHue=%23faf8f5&clientId=u1f08deda-8715-4&from=paste&height=465&id=ub17b762d&name=image.png&originHeight=780&originWidth=1080&originalType=binary&ratio=1&rotation=0&showTitle=false&size=111160&status=done&style=none&taskId=u36c1dc59-5857-4c8a-a3fb-6c298a1d8e7&title=&width=644)

1. Redis 父线程首先判断，当前是否在执行 save ，或 bgsave/bgrewriteof 的子进程，如果在执行则 bgsave 命令直接返回，bgsave/bgrewriteof 的子进程不能同时执行，主要是基于性能方面的考虑，两个并发的子进程同时执行大量的磁盘写操作，可能引发严重的性能问题
2. 父进程执行 fork 操作创建子进程，这个过程中父进程是阻塞的，Redis 不能执行来自客户端的任何命令。父进程 fork 后，bgsave 命令返回”Background saving started”信息并不再阻塞父进程，并可以响应其他命令
3. 子进程进程对内存数据生成快照文件。
4. 父进程在此期间接收的新的写操作，使用 COW 机制写入。
5. 子进程完成快照写入，替换旧 RDB 文件，随后子进程退出。
#### Fork 子线程的作用
关于fork

- Linux 操作系统中的程序，fork 会产生一个和父进程完全相同的子进程。子进程与父进程所有的数据均一致，但是子进程是一个全新的进程，与原进程是父子进程关系。
- 出于效率考虑，Linux 操作系统中使用 COW(Copy On Write)写时复制机制，fork 子进程一般情况下与父进程共同使用一段物理内存，只有在进程空间中的内存发生修改时，内存空间才会复制一份出来。

在 Redis 中，RDB 持久化就是充分的利用了这项技术，Redis 在持久化时调用 glibc 函数 fork 一个子进程，全权负责持久化工作，这样父进程仍然能继续给客户端提供服务。fork 的子进程初始时与父进程（Redis 的主进程）共享同一块内存；当持久化过程中，客户端的请求对内存中的数据进行修改，此时就会通过 COW (Copy On Write) 机制对数据段页面进行分离，也就是复制一块内存出来给主进程去修改
# AOF
AOF  (Append Only File)  是把所有对内存进行修改的指令（写操作）以独立日志文件的方式进行记录，重启时通过执行 AOF 文件中的 Redis 命令来恢复数据。类似MySql bin-log 原理。AOF 能够解决数据持久化实时性问题，是现在 Redis 持久化机制中主流的持久化方案。
优点：

- 数据的备份更加完整，丢失数据的概率更低，适合对数据完整性要求高的场景
- 日志文件可读，AOF可操作性更强，可通过操作日志文件进行修复

缺点：

- AOF 日志文件在长期运行中逐渐庞大，恢复起来非常耗时，需要定期对AOF文件进行瘦身处理
- 恢复备份的速度较慢
- 同步写操作频繁会带来性能压力
## AOF 文件内容
被写入 AOF 文件的所有命令都是以 RESP 格式保存的，是纯文本格式保存在 AOF 文件中。
> Redis 客户端和服务端之间使用一种名为 RESP(REdis Serialization Protocol) 的二进制安全文本协议进行通信。

## AOF 持久化实现
AOF 持久化方案进行备份时，客户端所有请求的写命令都会被追加到 AOF 缓冲区中，缓冲区中的数据会根据 Redis 配置文件中配置的同步策略来同步到磁盘上的 AOF 文件中，追加保存每次写的操作到文件末尾。同时当 AOF 的文件达到重写策略配置的阈值时，Redis 会对 AOF 日志文件进行重写，给 AOF 日志文件瘦身。Redis 服务重启的时候，通过加载 AOF 日志文件来恢复数据。
### 命令追加（append）
Redis 先将写命令追加到缓冲区 aof_buf，而不是直接写入文件，主要是为了避免每次有写命令都直接写入硬盘，导致硬盘 IO 成为 Redis 负载的瓶颈。
### 文件写入（write）和文件同步（sync）
Linux 操作系统中为了提升性能，使用了页缓存（page cache）。当我们将 aof_buf 的内容写到磁盘上时，此时数据并没有真正的落盘，而是在 page cache 中，为了将 page cache 中的数据真正落盘，需要执行 fsync / fdatasync 命令来强制刷盘。这边的文件同步做的就是刷盘操作，或者叫文件刷盘可能更容易理解一些。
AOF 缓存区的同步文件策略由参数 appendfsync 控制，有三种同步策略，各个值的含义如下：

- `always`：命令写入 aof_buf 后立即调用系统 write 操作和系统 fsync 操作同步到 AOF 文件，fsync 完成后线程返回。这种情况下，每次有写命令都要同步到 AOF 文件，硬盘 IO 成为性能瓶颈，Redis 只能支持大约几百TPS写入，严重降低了 Redis 的性能；即便是使用固态硬盘（SSD），每秒大约也只能处理几万个命令，而且会大大降低 SSD 的寿命。可靠性较高，数据基本不丢失。
- `no`：命令写入 aof_buf 后调用系统 write 操作，不对 AOF 文件做 fsync 同步；同步由操作系统负责，通常同步周期为30秒。这种情况下，文件同步的时间不可控，且缓冲区中堆积的数据会很多，数据安全性无法保证。
- `everysec`：命令写入 aof_buf 后调用系统 write 操作，write 完成后线程返回；fsync 同步文件操作由专门的线程每秒调用一次。everysec 是前述两种策略的折中，是性能和数据安全性的平衡，因此是 Redis 的默认配置，也是我们推荐的配置。
### 文件重写
主要作用是压缩 AOF 文件
AOF 重写是 AOF 持久化的一个机制，用来压缩 AOF 文件，通过 fork 一个子进程，重新写一个新的 AOF 文件，**该次重写不是读取旧的 AOF 文件进行复制，而是读取内存中的 Redis 数据库**，重写一份 AOF 文件，有点类似于 RDB 的快照方式。
之所以能压缩，原因是：

- 过期的数据不会再写入文件
- 无效的命令不再写入文件：如有些数据被重复设值(set mykey v1, set mykey v2)、有些数据被删除了(sadd myset v1, del myset)等等
- 多条命令可以合并为一个：如 sadd myset v1, sadd myset v2, sadd myset v3 可以合并为 sadd myset v1 v2 v3。不过为了防止单条命令过大造成客户端缓冲区溢出，对于 list、set、hash、zset类型的 key，并不一定只使用一条命令；而是以某个常量为界将命令拆分为多条。这个常量在 redis.h/REDIS_AOF_REWRITE_ITEMS_PER_CMD 中定义，不可更改，2.9版本中值是64。
# 混合持久化
Redis4.0 后大部分的使用场景都不会单独使用 RDB 或者 AOF 来做持久化机制，而是兼顾二者的优势混合使用。其原因是 RDB 虽然快，但是会丢失比较多的数据，不能保证数据完整性；AOF 虽然能尽可能保证数据完整性，但是性能确实是一个诟病，比如重放恢复数据。
Redis 4.0之后，引入 RDB-AOF 混合持久化模式，这种模式是基于 AOF 持久化模式构建而来的，混合持久化通过` aof-use-rdb-preamble yes`开启。

![image.png](https://cdn.nlark.com/yuque/0/2022/png/28316065/1666942830524-296d2812-5aec-4daf-98f8-0021d63fd5e8.png#averageHue=%23fbfaf8&clientId=u1f08deda-8715-4&from=paste&height=279&id=ucb292a3d&name=image.png&originHeight=427&originWidth=1080&originalType=binary&ratio=1&rotation=0&showTitle=false&size=38889&status=done&style=none&taskId=u94b98f91-5246-4180-94e9-3cb65f12e6d&title=&width=706)

