# String
String 类型是简单上手的数据类型，也是很多复杂类型的基础，因为 List 类型是 String 类型的链表，Set 类型是 String 类型的 Set，等等

- Create：SET
- Retreive：GET
- Update：SET
- Delete：DEL
- Others：INCR，DECR

上述操作都是 O(1) 的时间复杂度，其实 Redis 存储的类型都是 String 类型，即使是像 INCR 这样的操作也是先将 String 转化成 Integer 之后再进行计算
# Lists
Redis 的 LIst 是通过链表来实现的。

- Create：LPUSH，RPUSH
- Delete：LPOP，RPOP
- Others：LLEN
- Retrieve：LINDEX，LRANGE
- Update：LSET

其中对于 LLEN 操作，其时间复杂度其实是 O(1)，因为 List 结构存储了长度这个字段，每次修改的时候会对这个字段进行修改
## Use case
想象一下，您的主页显示了照片共享社交网络中发布的最新照片，并且您想加快访问速度。每次用户发布新照片时，我们都会使用 LPUSH 将其 ID 添加到列表中。 当用户访问主页时，我们使用 LRANGE 0 9 以获得最新的 10 个发布项目。
# Sets
Set 存储的是不重复的，没有顺序的 String 元素集合。
如果有一个项目集合，并且希望以非常快速的方式检查集合的存在性 (SISMEMBER) 或集合的大小 (SCARD)， 那么 Set 是首选，Set 还支持 peek 元素 (SRANDMEMBER) 或弹出随机元素 (SPOP)。

- Create：SADD
- Delete：SREM，SPOP
- Retrieve：SISMEMBER
- Others：SCARD，SDIFF

Set 很适合处理集合之间的关系，比如并集，交集等
# Hash
Redis Hashes 是 key-value 映射，非常适合用于表示对象

- Create：HSET
- Retrieve：HGET
- Others：HLEN
# Sorted Sets
Redis Sorted Sets 与 Redis Sets 类似，是字符串的非重复集合。 不同之处在于，Sorted Set 的每个成员都与一个 score 相关联，该分数用于从最小到最大 score 对排序集进行排序。
适合用于游戏排行榜之类 use case
## ZSET 的底层数据结构是跳表，为什么不用红黑树？

1. 跳表的实现更加简单，不用旋转节点，相对效率更高
2. 跳表在进行范围查询的时候效率是高于红黑树的，因为跳表只需要往后连续遍历，而红黑树需要中序遍历
3. 平衡树的插入和删除可能引发子树的调整，逻辑较为复杂，而跳表只需要维护相邻节点
4. 从内存占用的角度来看，跳表比红黑树更加灵活，一般来说，平衡树的每个节点包含 2 个指针，而跳表每个节点包含的平均指针数为 1/(1-p) ，

