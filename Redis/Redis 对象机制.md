# 为什么 Redis 会设计 redisObject 对象
在 redis 的命令中，用于对键进行处理的命令占了很大一部分，而对于键所保存的值的类型（键的类型），键能执行的命令又各不相同，如：`LPUSH`和`LLEN`只能用于列表键，而`SADD`和`SRANDMEMBER`只能用于集合键，等等，想要正确实现这些命令，必须为不同的类型的键设置不同的处理方式，比如删除一个列表键和删除一个字符串键的操作过程就不太一样

所以，**Redis 必须让每个键都带有类型信息, 使得程序可以检查键的类型, 并为它选择合适的处理方式，操作数据类型的命令除了要对键的类型进行检查之外, 还需要根据数据类型的不同编码进行多态处理，**为此，Redis 构建了自己的类型系统

- redisObject 对象
- 基于 redisObject 对象的类型检查
- 基于 redisObject 对象的显式多态函数
- 对 redisObject 进行分配、共享和销毁的机制
# redisObject 数据结构
```c
/*
* Redis 对象
*/
typedef struct redisObject {

    // 类型
    unsigned type:4;

    // 编码方式
    unsigned encoding:4;

    // LRU - 24位, 记录最末一次访问时间（相对于lru_clock）; 或者 LFU（最少使用的数据：8位频率，16位访问时间）
    unsigned lru:LRU_BITS; // LRU_BITS: 24

    // 引用计数
    int refcount;

    // 指向底层数据结构实例
    void *ptr;

} robj;
```
其中，type、encoding 和 ptr 是最重要的三个属性

- type 记录了对象所保存的值的类型
   - #define OBJ_STRING 0 // 字符串
   - #define OBJ_LIST 1 // 列表
   - #define OBJ_SET 2 // 集合
   - #define OBJ_ZSET 3 // 有序集
   - #define OBJ_HASH 4 // 哈希表
- encoding 记录了对象所保存的值的编码
   - #define OBJ_ENCODING_RAW 0     /* Raw representation */
   - #define OBJ_ENCODING_INT 1     /* Encoded as integer */
   - #define OBJ_ENCODING_HT 2      /* Encoded as hash table */
   - #define OBJ_ENCODING_ZIPMAP 3  /* 注意：版本2.6后不再使用. */
   - #define OBJ_ENCODING_LINKEDLIST 4 /* 注意：不再使用了，旧版本2.x中String的底层之一. */
   - #define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
   - #define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
   - #define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
   - #define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
   - #define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
   - #define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
- ptr 是一个指针，指向实际保存值的数据结构，这个数据结构由 type 和 encoding 属性决定，例如，如果一个 redisObject 的 type 属性为`OBJ_LIST`，encoding 属性为`OBJ_ENCODING_QUICKLIST`，那么这个对象就是一个 Redis 列表（List），它的值保存在一个 QuickList 的数据结构内，而 ptr 指针就指向 quicklist 对象

![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1688458631413-b1c11fa4-0b81-4a92-be35-d91e112b592f.png#averageHue=%23f1ece0&clientId=u597a0fd4-17a8-4&from=paste&height=540&id=u5b3e5e96&originHeight=1128&originWidth=1938&originalType=binary&ratio=2.5&rotation=0&showTitle=false&size=255920&status=done&style=none&taskId=uf385cc0a-6253-4c03-92d4-fd9b8ca3d46&title=&width=927.2000122070312)

- lru 记录了对象最后一次被命令程序访问的时间
# 命令的类型检查和多态

1. 根据给定的 key，在数据库字典中查找和它相对应的 redisObject，如果没找到，则返回 NULL
2. 检查 redisObject 的 type 属性和执行命令所需的类型是否相符，如果不相符，返回类型错误
3. 根据 redisObject 的 encoding 属性所指定的编码，选择合适的操作函数来处理底层的数据结构
4. 返回数据结构的操作结果作为命令的返回值
# 对象共享
> redis 一般会把一些常见的值放到一个共享对象中，这样可使程序避免了重复分配的麻烦，也节约了一些 CPU 时间

redis 预分配的值对象如下：

- 各种命令的返回值，比如成功时返回的 OK，错误时返回的 ERROR，命令入队事务时返回的 QUEUE 等
- 包括 0 在内，小于`REDIS_SHARED_INTEGERS`的所有整数（默认值是10000）
> 注意：共享对象只能被字典和双向链表这类能带有指针的数据结构使用。像整数集合和压缩列表这些只能保存字符串、整数等自勉之的内存数据结构

# 引用计数以及对象的销毁
> redisObject 中有 refcount 属性，是对象的引用计数，显然计数 0 那么就是可以回收了

