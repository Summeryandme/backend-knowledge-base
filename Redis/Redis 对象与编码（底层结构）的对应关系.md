![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1688955134290-89c5ffd6-2119-443c-92c8-9ca229350b24.png#averageHue=%23f2ede2&clientId=u98c7f40d-0444-4&from=paste&height=555&id=u7b531f21&originHeight=1128&originWidth=1938&originalType=binary&ratio=2&rotation=0&showTitle=false&size=275952&status=done&style=none&taskId=u309f3f20-5f21-4754-9281-43c758f2a13&title=&width=954.2000122070312)
# 字符串对象
> 字符串是Redis最基本的数据类型，不仅所有key都是字符串类型，其它几种数据类型构成的元素也是字符串。注意字符串的长度不能超过**512M**。

## 编码
字符串对象的编码可以是 int，raw 或者 embstr

- int，保存的是可以用 long 类型表示的整数值
- embstr，保存长度小于 44 字节的字符串
- raw，保存长度大于 44 字节的字符串
## 内存布局
三种内存布局如下图所示
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1688955447333-884b8975-d9fe-4e70-94b8-704544d904eb.png#averageHue=%23c1b546&clientId=u98c7f40d-0444-4&from=paste&height=562&id=uca56d41d&originHeight=517&originWidth=708&originalType=binary&ratio=2&rotation=0&showTitle=false&size=26225&status=done&style=none&taskId=u82f062ab-a37e-4615-89e0-97dbb5326c1&title=&width=769.2000122070312)
## raw 和 embstr 的区别
embstr 的使用只分配一次内存空间（因此 redisObject 和 sds 是连续的），而 raw 需要分配两次内存空间（分别为 redisObject 和 sds 分配空间），因此与 raw 相比，embstr 的好处在于创建时少分配一次空间，删除时少释放一次空间，以及对象的所有数据连在一起，寻找方便。而embstr 的坏处也很明显，如果字符串的长度增加需要重新分配内存时，整个 redisObject 和 sds 都需要重新分配空间，因此 redis 中的 embstr 实现为只读。
## 编码的转换

- 当 int 编码保存的值不再是整数时，或大小超过 long 的范围时，自动转化为 raw
- 对于 embstr 编码，由于 Redis 没有对其编写任何的修改程序（embstr 为只读），在对 embstr 对象进行修改时，都会先转化为 raw 再进行修改，因此，只要是修改 embstr 对象，修改之后都会变成 raw
# 列表对象
> list 列表，它是简单的字符串列表，按照插入顺序排序，你可以添加一个元素到列表的头部（左边）或者尾部（右边），它的底层实际上是个链表结构。

## 编码
quicklist
## 内存布局
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1688956713250-7b323585-05db-45b1-88ef-77025b28a675.png#averageHue=%23fcf7f3&clientId=u98c7f40d-0444-4&from=paste&height=509&id=u73edc2e4&originHeight=609&originWidth=1109&originalType=binary&ratio=2&rotation=0&showTitle=false&size=44398&status=done&style=none&taskId=u330c65e9-696e-42bb-9578-e7a73d5b37f&title=&width=926.6000061035156)
# 哈希对象
## 编码
哈希对象的编码可以是 ziplist 或者 hashtable，对应的底层实现有两种, 一种是 ziplist, 一种是 dict。
## 内存布局
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1688956801979-ac9b735b-e335-4a16-a271-faead549d136.png#averageHue=%23f6f0ec&clientId=u98c7f40d-0444-4&from=paste&height=585&id=u6f4018b3&originHeight=714&originWidth=1141&originalType=binary&ratio=2&rotation=0&showTitle=false&size=71998&status=done&style=none&taskId=u0efb851a-7ec8-45e6-b6c3-23226471225&title=&width=934.3999938964844)
上图不严谨的地方在于

1. ziplist 中每 个entry, 除了键与值本身的二进制数据, 还包括其它字段, 图中没有画出来
2. dict 底层可能持有两个 dictht 实例
3. 没有画出 dict 的哈希冲突
## 编码转换
和上面列表对象使用 ziplist 编码一样，当同时满足下面两个条件时，使用 ziplist（压缩列表）编码：

1. 列表保存元素个数小于 512 个
2. 每个元素长度小于 64 字节

不能满足这两个条件的时候使用 hashtable 编码。以上两个条件也可以通过 Redis 配置文件`zset-max-ziplist-entries` 选项和 `zset-max-ziplist-value` 进行修改。
# 集合对象
> 集合对象 set 是 string 类型（整数也会转换成 string 类型进行存储）的无序集合。注意集合和列表的区别：集合中的元素是无序的，因此不能通过索引来操作元素；集合中的元素不能有重复。

## 编码
集合对象的编码可以是 intset 或者 hashtable，底层实现有两种, 分别是 intset 和 dict。 显然当使用 intset 作为底层实现的数据结构时，集合中存储的只能是数值数据，且必须是整数，而当使用 dict 作为集合对象的底层实现时，是将数据全部存储于dict的键中，值字段闲置不用
## 内存布局
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1688957436092-7ebcc243-1f0e-4cbd-a89c-70b8560f499b.png#averageHue=%23f5eeeb&clientId=u98c7f40d-0444-4&from=paste&height=486&id=u458d10fa&originHeight=591&originWidth=1138&originalType=binary&ratio=2&rotation=0&showTitle=false&size=66236&status=done&style=none&taskId=u58aa7f8f-eb96-428a-bb36-0d2b64c0afc&title=&width=935.2000122070312)
# 有序集合对象
> 和上面的集合对象相比，有序集合对象是有序的。与列表使用索引下标作为排序依据不同，有序集合为每个元素设置一个分数（score）作为排序依据。

## 编码
有序集合的底层实现依然有两种，一种是使用 ziplist 作为底层实现, 另外一种比较特殊, 底层使用了两种数据结构: dict 与 skiplist. 前者对应的编码值宏为 ZIPLIST, 后者对应的编码值宏为 SKIPLIST
## 内存布局
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1688957586266-53a859e6-5689-4817-943f-f40baf0be8a7.png#averageHue=%2385b849&clientId=u98c7f40d-0444-4&from=paste&height=163&id=u6d091be1&originHeight=119&originWidth=587&originalType=binary&ratio=2&rotation=0&showTitle=false&size=9404&status=done&style=none&taskId=u3ecfcc6f-6c10-4447-acff-b5f53233fce&title=&width=805.8000030517578)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1688957603064-a66dedd3-fac4-4ef0-a58c-563939ff426e.png#averageHue=%23f6eeec&clientId=u98c7f40d-0444-4&from=paste&height=599&id=u7fe600d8&originHeight=854&originWidth=1148&originalType=binary&ratio=2&rotation=0&showTitle=false&size=79335&status=done&style=none&taskId=ub2b0eae6-ff45-4dbe-a268-d533d06be5c&title=&width=805.2000122070312)

