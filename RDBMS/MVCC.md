# MVCC
即 Multi-Version Concurrency Control
数据库在存储一行数据记录时，会分为两个部分进行存储
- 数据行的额外信息
- 真实的数据记录

针对额外信息，数据库会为每一行真实数据记录添加三个隐藏字段

- row_id：如果表中有自定义的主键，就不会添加此字段，否则，会自动生成
- transaction_id：事务 id，代表这一行数据是由哪个事务 id 创建的
   - 当开启一个事务时，并不会立马获取到事务 id，只有在执行 insert/update/delete 操作时，才会获取
- roll_pointer：回滚指针，指向这行数据的上一个版本

对于 READ CUNCOMMITED 隔离级别，可以读取到其他事务还没有提交的数据，所以直接把这个数据的最新版本读出来就行了，对于 SERIALIZABLE 隔离级别，是用加锁的方式来访问记录
剩下的就是 READ COMMITTED 和 REPEATABLE READ，这两个事务隔离级别都要保证读到的数据是其他事务已经提交的，也就是不能无脑把一行数据的最新版本给读出来了，但是这两个还是有一定的区别，最核心的问题就在于“到底可以读取这个数据的哪个版本”
## ReadView
它包含四个比较重要的内容

- m_ids：表示在生成 ReadView 时，系统中活跃的事务 id 的集合
- min_trx_id：表示在生成 ReadView 时，系统中活跃的最小的事务 id，也就是 m_ids 中的最小值
- max_trx_id：表示在生成 ReadView 时，系统应该分配给下一个事务的 id
- creator_trx_id：表示生成该 ReadView 的事务 id

如何使用这个 ReadView？

- 如果被访问的版本的 trx_id 和 ReadView 中的 creator_trx_id 相同，就意味着当前版本就是由你“造成”的，可以读出来
- 如果被访问的版本的 trx_id 小于 ReadView 中的 min_trx_id，表示生成该版本的事务在创建 ReadView 的时候，已经提交了，所以该版本可以读出来
- 如果被访问的版本的 trx_id 大于或等于 ReadView 中的 max_trx_id 值，说明生成该版本的事务在当前事务生成ReadView后才开启，所以该版本不可以被读出来
- 如果被访问的版本的 trx_id 在 min_trx_id 和 max_trx_id 之间，那就需要判断下trx_id在不在m_ids中：如果在，说明创建 ReadView 的时候，生成该版本的事务还是活跃的（没有被提交），该版本不可以被读出来；如果不在，说明创建 ReadView 的时候，生成该版本的事务已经被提交了，该版本可以被读出来

如果某个数据的最新版本不可以被读出来，就顺着roll_pointer找到该数据的上一个版本，继续做如上的判断，以此类推，如果第一个版本也不可见的话，代表该数据对当前事务完全不可见，查询结果就不包含这条记录了
READ COMMITED 和 REPEATABLE READ 的区别就是：

- 前者每次读取数据的时候都会创建一个 ReadView
- 后者只会在首次读取数据的时候创建 ReadView