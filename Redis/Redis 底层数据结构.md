# SDS - 简单动态字符串
```cpp
// 注意：sdshdr5从未被使用，Redis中只是访问flags。
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 低3位存储类型, 高5位存储长度 */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* 已使用 */
    uint8_t alloc; /* 总长度，用1字节存储 */
    unsigned char flags; /* 低3位存储类型, 高5位预留 */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* 已使用 */
    uint16_t alloc; /* 总长度，用2字节存储 */
    unsigned char flags; /* 低3位存储类型, 高5位预留 */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* 已使用 */
    uint32_t alloc; /* 总长度，用4字节存储 */
    unsigned char flags; /* 低3位存储类型, 高5位预留 */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* 已使用 */
    uint64_t alloc; /* 总长度，用8字节存储 */
    unsigned char flags; /* 低3位存储类型, 高5位预留 */
    char buf[];
};
```

- len：SDS 保存的字符串的长度
- alloc（被分配了多少字节）： 分别以 uint8, uint16, uint32, uint64 表示整个 SDS, 除过头部与末尾的 \0, 剩余的字节数
- flags：始终为一字节，以低 3 位标示着头部的类型，高 5 位未使用
## 为什么要使用 SDS

- 常数复杂度获得字符串长度

由于 len 属性的存在，我们获取 SDS 字符串的长度只需要读取 len 属性，时间复杂度为 O(1)。而对于 C 语言，获取字符串的长度通常是经过遍历计数来实现的，时间复杂度为 O(n)。通过 `strlen key` 命令可以获取 key 的字符串长度。

- 杜绝缓冲区溢出

我们知道在 C 语言中使用 strcat 函数来进行两个字符串的拼接，一旦没有分配足够长度的内存空间，就会造成缓冲区溢出。而对于 SDS 数据类型，在进行字符修改的时候，会首先根据记录的 len 属性检查内存空间是否满足需求，如果不满足，会进行相应的空间扩展，然后在进行修改操作，所以不会出现缓冲区溢出。

- 减少修改字符串的内存重新分配次数

C 语言由于不记录字符串的长度，所以如果要修改字符串，必须要重新分配内存（先释放再申请），因为如果没有重新分配，字符串长度增大时会造成内存缓冲区溢出，字符串长度减小时会造成内存泄露。
而对于SDS，由于 len 属性和 alloc 属性的存在，对于修改字符串 SDS 实现了空间预分配和惰性空间释放两种策略：

1. 空间预分配：对字符串进行空间扩展的时候，扩展的内存比实际需要的多，这样可以减少连续执行字符串增长操作所需的内存重分配次数。
   1. 当新字符串的长度小于 1M 时，Redis 会分配他们所需大小一倍的空间
   2. 当新字符串的长度大于 1M 时，Redis 会给它 +1M
2. 惰性空间释放：对字符串进行缩短操作时，程序不立即使用内存重新分配来回收缩短后多余的字节，而是使用 alloc 属性将这些字节的数量记录下来，等待后续使用。（当然 SDS 也提供了相应的 API，当我们有需要时，也可以手动释放这些未使用的空间。）
- 二进制安全

因为 C 字符串以空字符作为字符串结束的标识，而对于一些二进制文件（如图片等），内容可能包括空字符串，因此 C 字符串无法正确存取；而所有 SDS 的 API 都是以处理二进制的方式来处理 buf 里面的元素，并且 SDS 不是以空字符串来判断是否结束，而是以 len 属性表示的长度来判断字符串是否结束。

- 兼容部分 C 字符串函数

虽然 SDS 是二进制安全的，但是一样遵从每个字符串都是以空字符串结尾的惯例，这样可以重用 C 语言库`<string.h>` 中的一部分函数。
# ZipList - 压缩列表
整个 ziplist 在内存中的存储格式如下
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1688521917745-138e6fa9-ddaf-4a0a-ba08-de5cf5efaea2.png#averageHue=%23e1dbd4&clientId=u56bd8d47-cb84-4&from=paste&height=56&id=uc750e137&originHeight=57&originWidth=763&originalType=binary&ratio=2.5&rotation=0&showTitle=false&size=7803&status=done&style=none&taskId=u3522df89-42e3-4274-b823-aaaca535d62&title=&width=752.2000122070312)

- `zlbytes`字段的类型是 unit32_t，这个字段中存储的是整个 ziplist 所占用的内存的字节数
- `zltail`字段的类型是 unit32_t，它指的是 ziplist 最后一个 entry 的偏移量，用于快速定位最后一个 entry，以快速完成 pop 等操作
- `zllen`字段的类型是 uint16_t，它指的是整个 ziplist 中 entry 的容量，这个值只占 2bytes，如果数量小于 65525（2 的 16 次方 - 1），字段就是实际含义的值，如果大于，则固定为 65525，实际的值则需通过遍历去获得
- `zlend`是一个终止字节，其值为全 f，ziplist 保证任何情况下, 一个 entry 的首字节都不会是255
## entry 的结构

- 一般结构：<prevlen> <encoding> <entry-data>
   - prevlen：前一个 entry 的大小
   - encoding：不同情况下的值不同，用于表示当前 entry 的类型和长度
   - entry-data：真实用于存储 entry 表示的数据
- 在 entry 中存储的是 int 类型时，encoding 和 entry-data 会合并在 encoding 中表示，此时没有 entry-data 字段，在 redis 中，在存储数据时，会先尝试将 string 转换成 int 存储，节省空间
## 优点

- ziplist 节省内存是相对于普通的 list 来说的，如果是普通的数组，那么它每个元素占用的内存是一样的且取决于最大的那个元素（很明显它是需要预留空间的）
- 所以 ziplist 在设计时就很容易想到要尽量让每个元素按照实际的内容大小存储，所以增加 encoding 字段，针对不同的 encoding 来细化存储大小
- 这时候还需要解决的一个问题是遍历元素时如何定位下一个元素呢？在普通数组中每个元素定长，所以不需要考虑这个问题；但是 ziplist 中每个 data 占据的内存不一样，所以为了解决遍历，需要增加记录上一个元素的 length，所以增加了 prelen 字段
## 缺点

- ziplist 也不预留内存空间, 并且在移除结点后, 也是立即缩容, 这代表每次写操作都会进行内存分配操作
- 结点如果扩容, 导致结点占用的内存增长, 并且超过254字节的话, 可能会导致链式反应: 其后一个结点的entry.prevlen 需要从一字节扩容至五字节. 最坏情况下, 第一个结点的扩容, 会导致整个 ziplist 表中的后续所有结点的 entry.prevlen 字段扩容. 虽然这个内存重分配的操作依然只会发生一次, 但代码中的时间复杂度是o(N)级别, 因为链式扩容只能一步一步的计算. 但这种情况的概率十分的小, 一般情况下链式扩容能连锁反映五六次就很不幸了. 之所以说这是一个蛋疼问题, 是因为, 这样的坏场景下, 其实时间复杂度并不高: 依次计算每个entry 新的空间占用, 也就是o(N), 总体占用计算出来后, 只执行一次内存重分配, 与对应的 memmove 操作, 就可以了
# QuickList - 快表
它是一个以 ziplist 为结点的双端链表结构，宏观上，quicklist 是一个链表，微观上，链表中每个结点都是一个 ziplist
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1688547200967-082b27e9-84c8-4ab4-8300-0748b6624ad4.png#averageHue=%23c8fcde&clientId=u6ec18c55-96b9-4&from=paste&height=560&id=uefcc18b4&originHeight=593&originWidth=884&originalType=binary&ratio=2.5&rotation=0&showTitle=false&size=36935&status=done&style=none&taskId=uac31660f-4bb9-4d4c-9e60-4a355e25765&title=&width=834.6000061035156)
```cpp
typedef struct quicklistNode {
    struct quicklistNode *prev; //上一个node节点
    struct quicklistNode *next; //下一个node
    unsigned char *zl;     // 保存的数据 压缩前ziplist 压缩后压缩的数据，如果当前节点的数据没有压缩，那么它指向一个ziplist结构；否则，它指向一个quicklistLZF结构
    unsigned int sz;             /* ziplist size in bytes */
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;
```
## 额外信息

- `quicklist.fill`的值影响着每个链表结点中，ziplist 的长度
   - 当数值为负数时，代表以字节数限制单个 ziplist 的最大长度，例如 -1 表示不超过 4kb，-2 表示不超过 8kb
   - 当数值为正数时，代表以 entry 数目限制单个 ziplist 的长度
- `quicklist.compress`的值影响着 quicklistNode.zl 字段指向的是原生的 ziplist，还是经过压缩包装后的 quicklistLZF
   - 0 表示不压缩
   - 1 表示 quicklist 的链表头尾结点不压缩，其余结点的 zl 字段指向的是经过压缩后的 quicklistLZF
   - 2 表示 quicklist 的链表头两个, 与末两个结点不压缩, 其余结点的zl字段指向的是经过压缩后的 quicklistLZF
   - 。。。以此类推
- `quicklistNode.encoding`字段, 以指示本链表结点所持有的 ziplist 是否经过了压缩. 1 代表未压缩, 持有的是原生的 ziplist,  2 代表压缩过
- `quicklistNode.container`字段指示的是每个链表结点所持有的数据类型是什么. 默认的实现是 ziplist, 对应的该字段的值是 2, 目前Redis 没有提供其它实现. 所以实际上, 该字段的值恒为 2
- `quicklistNode.recompress`字段指示的是当前结点所持有的 ziplist 是否经过了解压. 如果该字段为1即代表之前被解压过, 且需要在下一次操作时重新压缩.
# Dict - 字典/哈希表
其实就是一个传统意义上的**哈希表**

- 解决哈希冲突使用的是链地址法
- 扩容和收缩
   - 如果执行扩展操作，会基于原哈希表创建一个大小等于`ht[0].used * 2n`的哈希表，相反如果是收缩操作，则缩小一倍
   - 重新利用刚创建的新哈希表，计算索引值，然后将键值对放到新的哈希表的位置上
   - 所有键值对都迁移完毕之后，释放原哈希表的空间
- 触发扩容的条件
   - 目前没有执行`bgsave`或者`bgrewriteof`，并且负载因子大于 1
   - 正在执行`bgsave`或者`bgwriteof`，并且负载因子大于 5
   - 负载因子 = 已经保存节点的数量 / 哈希表大小
- 渐进式 rehash：字典的删除/查找/更新等操作可能会在两个哈希表上进行，第一个哈希表没有找到，就会去第二个哈希表上进行查找。但是进行 增加操作，一定是在新的哈希表上进行的
# IntSet - 整数集
> 整数集合（intset）是集合类型的底层实现之一，当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis 就会使用整数集合作为集合键的底层实现。

## 结构
```cpp
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```

- encoding：表示编码方式，取值有三个：`INTSET_ENC_INT16`, `INTSET_ENC_INT32`, `INTSET_ENC_INT64`
- length：代表其中存储的整数的个数
- contents：指向实际存储数值的连续内存区域，就是一个数组，整数集合的每个元素都是 contents 数组的一个数组项（item），各个项在数组中按值得大小从小到大有序排序，且数组中不包含任何重复项。（虽然 intset 结构将 contents 属性声明为 int8_t 类型的数组，但实际上 contents 数组并不保存任何 int8_t 类型的值，contents 数组的真正类型取决于 encoding 属性的值）
## 升级
当在一个 int16 类型的整数集合中插入一个 int32 类型的值，整个集合的所有元素都会转换成32类型。 整个过程有三步：

- 根据新元素的类型，扩展整数集合底层数组的空间大小，并为新元素分配空间
- 将底层数组现有的所有元素都转换成与新元素相同的类型， 并将类型转换后的元素放置到正确的位上， 而且在放置元素的过程中， 需要继续维持底层数组的有序性质不变
- 最后改变 encoding 的值，length + 1

暂时不存在**降低**的操作
# ZSkipList - 跳表
> 跳跃表结构在 Redis 中的运用场景只有一个，那就是作为有序列表 (Zset) 的使用。跳跃表的性能可以保证在查找，删除，添加等操作的时候在对数期望时间内完成，这个性能是可以和平衡树来相比较的，而且在实现方面比平衡树要优雅，这就是跳跃表的长处。跳跃表的缺点就是需要的存储空间比较大，属于利用空间来换取时间的数据结构。

- 头结点不持有任何数据，且其 level[] 的长度为32
- 每个结点
   - ele：持有数据，是 sds 类型
   - score：其标示着结点的得分，结点之间凭借得分来判断先后顺序，跳跃表中的结点按结点的得分升序排列
   - backward：指针，这是原版跳跃表中所没有的，该指针指向结点的前一个紧邻结点
   - level[]：
      - forward 字段指向比自己得分高的某个结点（不一定是紧邻的），并且, 若当前 zskiplistLevel 实例在 level[] 中的索引为 X, 则其 forward 字段指向的结点, 其 level[] 字段的容量至少是 X+1. 这也是上图中, 为什么 forward 指针总是画的水平的原因
      - span 字段代表 forward 字段指向的结点, 距离当前结点的距离. 紧邻的两个结点之间的距离定义为 1






