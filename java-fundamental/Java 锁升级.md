# 为什么线程切换很慢？
其实所谓用户态和内核态的切换只是其中一个方面，在多核 CPU 的情况下，当你切换了线程，由于重新触发的时候可能这个线程已经在另一个 CPU 上了，于是意味着线程在原先 CPU 上的缓存也许已经不再有用了
所以不建议没事就把线程挂起，而是让线程自旋比挂起要好很多
也正是这点，才有了后面的 sync 锁升级机制，理解了为什么要锁升级，才能逐步理解锁升级过程
# 对象头中的 mark-word
java 每个对象的对象头中，都有 32 或者 64 位的 mark-word
![image.png](https://cdn.nlark.com/yuque/0/2022/png/28316065/1667294433222-c0d39830-11b3-4f1a-94b1-52f54fde03ac.png#averageHue=%23cee1cb&clientId=u68af70fb-c096-4&from=paste&height=272&id=u2875a735&name=image.png&originHeight=338&originWidth=1058&originalType=binary&ratio=1&rotation=0&showTitle=false&size=147342&status=done&style=none&taskId=u7414480f-7f8a-498b-9df1-56ca7e4001f&title=&width=852.9090881347656)
## 锁状态标志位 + 偏向锁标记位 （2 bit + 1 bit）
除了 markword 中的 2 位锁状态标志位， 其他 62 位都会随着锁状态标志位的变化而变化
锁状态标志位：

- 01：无锁或者偏向锁状态，因此还需要额外的偏向锁标记位 1bit 来确定是无锁还是偏向锁
- 00：轻量级锁
- 10：重量级锁
- 11：已经被 GC 标记，即将释放

为何无锁/偏向锁的标志位是 01，而轻量级锁的标志位是 00？
答：如果将轻量级锁标志位设置为 00， 那么在判断标志位为 00 后，** m 无需再额外做一次 markWord >> 2 的操作，而是直接将 markWord 拿来当作地址使用即可！**
## hashcode（31 bit）
### markword 中的 hashcode 是哪个方法生成的？
实际上， markword 中的 hashcode 只由底层 JDK C++ 源码计算得到（ java侧调用方法为 System.identityHashCode() ）， 生成后固化到 markword 中。
### markword 中的 hashcode 是什么时候生成？
很容易误以为会是对象一创建就生成了。
实际上，是采用了**延迟加载**技术，只有在用到的时候才生成。
毕竟有可能对象创建出来使用时，并不需要做哈希的操作。
## gc 分代年龄（4 bit）
在 jvm 垃圾收集机制中， 决定年轻代什么时候进入老年代的根据之一， 就是确认他的分代年龄是否达到阈值，
分代年龄只有4bit可以看出，最大值只能是 15。因此我们设置的进入老年代年龄阈值 -XX:MaxTenuringThreshold 最大只能设置 15。
## cms_free（1 bit）
实际上就是只有 CMS 收集器用到的。但最新 java11 中更多用的是 G1 收集器了，这一位相当于不怎么常用，因此提到的也非常少。
# 锁升级的四个阶段
## 无锁
markword如下：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/28316065/1667295736427-a8b5ad4d-ae13-470c-8262-77a172a798da.png#averageHue=%23efefef&clientId=u68af70fb-c096-4&from=paste&height=141&id=ubc983f32&name=image.png&originHeight=181&originWidth=1113&originalType=binary&ratio=1&rotation=0&showTitle=false&size=24683&status=done&style=none&taskId=u2abbc60e-9cb6-4024-91d3-01bd2726bab&title=&width=867.2000122070312)
无锁状态用于对象刚创建，且还未进入过同步代码块的时候
JVM 中有一个优化，对每一个新对象都预置一个可偏向状态，将最后三位设置为 101
注意此时 markword 中高位是不存在 ThreadID 的， 都是 0， 说明此时并没有线程偏向发生，因此也可以理解成是无锁。
好处在于后续做偏向锁加锁时，无需再去改动偏向锁标记位，只需要对线程 id 做 cas 即可。
## 偏向锁
一旦对象第一次进入 sync 同步方法块，就**可能**从无锁状态变成偏向锁状态
### 为什么要有偏向锁？
假设我们 new 出来的对象带有同步代码块方法，但在整个生命周期中只被一个线程访问，那么是否有必要做消耗的竞争动作，甚至引入额外的内存开销？没有必要。
因此针对的是 **对象有同步方法调用，但是实际不存在竞争的场景**
### markword 详解
markword 如下：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/28316065/1667296474897-bd43171e-d6ee-471f-aaf9-af5b93f193e2.png#averageHue=%23f2f2f2&clientId=u68af70fb-c096-4&from=paste&height=139&id=sQ3WY&name=image.png&originHeight=160&originWidth=975&originalType=binary&ratio=1&rotation=0&showTitle=false&size=16306&status=done&style=none&taskId=u967977fd-18a3-4665-b462-17890c94f38&title=&width=845)
#### 当前线程 id
这个 id 就是在进入了对象同步代码块的线程 id
**Java 的线程 id 是一个 long 类型，为什么这里只有 54 位？**
具体没有找到解释，可能是 jvm 团队认为 54 位线程id足够用了，不至于会创建 2^54 那么多的线程，真的有需要创建这么频繁的程序，也会优先采用线程池池才对
**线程 id 如何写入？**
线程id是直接写入markword吗？ 不对， 一定要注意到这时候是存在同时写的可能的。
因此会采用 CAS 的方式进行线程 id 的写入。 简而言之， 就是先取原线程 id 后，再更新线程 id，更新后检查一下是否和预期一致，不一致则说明被人改动过，则线程 id 写入失败，说明存在竞争，**升级为轻量级锁**。
轻量级锁是没有 hashcode 的，一旦在对象头中设置了 hashcode ，那么进入同步块的时候就不会进入偏向锁的状态，会直接跳到轻量级锁。
#### epoch是什么
通过 epoch，jvm 可以知道这个对象的偏向锁是否过期，过期的情况下允许直接试图抢占，而不进行撤销偏向锁的操作。
### 偏向锁运作详解
JVM 会通过 CAS 来设置偏向线程 id，一旦设置成功那么这个偏向锁就算是挂上了。
后面每次访问，检查线程 id 一致，就直接进入同步代码块了。
在偏向锁离开同步代码块的时候，这个偏向锁线程 id 并不会归零，后面只要有识别到一致的线程 id，就不用做特殊的处理。
当有其他线程来访问的时候，之前设置的偏向锁就有问题了，说明存在多线程访问同一个对象的情况
切换线程并不是一定会直接将偏向锁升级为轻量级锁，而是：

1. 当线程 B 发现是偏向锁，且线程 id 不是自己的时候，开始撤销操作
2. 首先，线程 B 会一直等待对象 obj 到达 JVM 安全点
3. 到达安全点之后，线程 B 检查线程 A 是否正处在 obj 的同步代码块内
4. 如果线程 A 正在同步代码块内，那么直接升级为轻量级锁
5. 如果线程 A 不在同步代码块内，那么线程 B 会先把偏向锁改成无锁状态，然后再用 CAS 的操作去尝试竞争，如果能竞争到，那么就会偏向自己
#### 批量重偏向，以及 epoch 的作用
假设有一个场景， 我们 new 了 30 个 obj 对象， 最初都是由 A 线程使用，后面通过 for 循环都由 B 线程使用，那么会发现在很短的时间内，连续发生了偏向锁撤销为无锁，且未因同步块竞争而发生轻量升级的情况。
那么，jvm 猜测此时后面都是类似的情况，于是 B 线程调用 obj 对象时，不再撤销了，直接 CAS 竞争 threadId，因为 jvm 预测 A 不会来抢了，具体步骤如下所示：

1. jvm 会在 obj 对象的类 class 对象中，定义一个偏向撤销计数器以及 epoch 偏向版本
2. 每当有一个对象被撤销偏向锁，都会让偏向撤销计数器 +1
3. 一旦加到 20，则认为出现大量的锁撤销动作，于是 class 类对象中的 epoch 值 +1（epoch 一般只有两位，0-3）
4. 接着，jvm 会找到所有正处在同步代码块中的 obj 对象，让他的 epoch 等于 class 类对象中的 epoch
5. 其他不在同步代码块中的 obj 对象，则不修改 epoch
6. 当 B 线程来访问时，发现 obj 对象的 epoch 和 class 对象的 epoch 不相等，则不再做撤销动作，直接 CAS 抢占，因为当 epoch 不相等的时候，说明该 obj 对象一直没被原主人使用，但它的兄弟们之前纷纷投降倒戈了，那我应该直接尝试占用就好了，没必要那么谨慎！
#### 批量撤销
但如果短时间内该类的撤销动作超过 40 个， jvm 会认为这个数量太多了，不保险，数量一多，预测就不准了。
jvm 此时会将 obj 对象的类 class 对象中的偏向标记（注意是类中的偏向锁开启标记，而不是对象头中的偏向锁标记）设置为禁用偏向锁。 后续该对象的 new 操作将直接走轻量级锁的逻辑。

即使你开启了偏向锁，但是这个偏向锁的启用是有延迟，大概 4s 左右。
**即 java 进程启动的 4s 内，都会直接跳过偏向锁，有同步代码块时直接使用轻量级锁。**
原因是 JVM 初始化的代码有很多地方用到了 synchronized，如果直接开启偏向，产生竞争就要有锁升级，会带来额外的性能损耗，jvm 团队经过测试和评估， 选择了启动速度最快的方案， 即强制 4s 内禁用偏向锁，所以就有了这个延迟策略 （当然这个延迟时间也可以通过参数自己调整）
偏向锁在JDK6引入, 且默认开启偏向锁优化, 可通过JVM参数-XX:-UseBiasedLocking来禁用偏向锁。
#### 偏向锁的重要演变历史
jdk 的演变过程中， 为偏向锁做了如上所述的批量升级、撤销等诸多动作。
但随着时代发展，发现偏向锁带来的维护、撤销成本， 远大于轻量级锁的少许 CAS 动作。
官方说明中有这么一段话: since the introduction of biased locking into HotSpot also change the amount of uncontended operations needed for that relation to remain true。
即随着硬件发展，原子指令成本变化，导致轻量级自旋锁需要的原子指令次数变少(或者cas操作变少 个人理解)，所以自旋锁成本下降，故偏向锁的带来的优势就更小了。
于是 jdk 团队在 jdk15 之后， 再次默认关闭了偏向锁。
还有就是奥卡姆剃刀原理， 如果增加的内容带来很大的成本，不如大胆的废除掉，接受一点落差，将精力放在提升度更大的地方。
## 轻量级锁
轻量级锁的 markword 如下所示，可以看到除了锁状态标记位以外，其他的都变成了一个栈帧中 lockRecord 的地址
![image.png](https://cdn.nlark.com/yuque/0/2022/png/28316065/1667375837352-04a6d17c-13cb-4bae-b8a3-7435f6d5a5b1.png#averageHue=%23f3f3f3&clientId=ucddd8009-ede9-4&from=paste&id=u05cba656&name=image.png&originHeight=167&originWidth=1152&originalType=url&ratio=1&rotation=0&showTitle=false&size=10782&status=done&style=none&taskId=u4be69401-5c3c-4eac-8628-dccf79613f9&title=)
原先的 markword 都去哪儿了？
之前提到 markword 中有分代年龄、cms_free、hashcode 等固有属性。
这些信息会被存储到对应线程栈帧中的 lockRecord 中。
lockRecord格式以及存储/交换过程如下：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/28316065/1667375937032-84516ad7-0a90-48a4-b91d-2399bc824b8b.png#averageHue=%23f3f3f3&clientId=ucddd8009-ede9-4&from=paste&id=u03b6ea96&name=image.png&originHeight=658&originWidth=800&originalType=url&ratio=1&rotation=0&showTitle=false&size=41231&status=done&style=none&taskId=ud75ae1fc-2753-424e-823d-48b42c6fd0b&title=)

**另外注意， 当轻量级锁未上锁时， 对象头中的 markword 存储的还是 markword 内容，并没有变成指针，只有当上锁过程中，才会变成指针。**
因此轻量级锁是存在反复的加锁和解锁操作的，解锁的过程就是通过 CAS 将对象头替换进去
### 线程重入问题
对于同一个线程，如果反复进入同步块，在 sync 语义上来说是支持重入的（即持有锁的线程可以多次进入锁区域）， 对轻量级锁而言，必须实现这个功能。
因此线程的 lockRecord 并非单一成员，他其实是一个 lockRecord 集合，可以存储多个 lockRecord。
每当线程离开同步块，lockRecord减少1个， 直到这个lockReocrd中包含指针，才会做解锁动作。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/28316065/1667377474549-c3aa3a8c-1d6b-4194-9640-9edd723724c9.png#averageHue=%23f3f3f3&clientId=ucddd8009-ede9-4&from=paste&id=u31ab7511&name=image.png&originHeight=694&originWidth=727&originalType=url&ratio=1&rotation=0&showTitle=false&size=41350&status=done&style=none&taskId=u6f672e05-382d-4cbd-bc8d-811c75dab74&title=)
### 加锁过程

1. 进入同步块之间，检查是否已经储存了 lockRecord 地址，且地址和自己当前线程一致，如果已经存了且一致，说明正处于重入操作，走重入逻辑，新增 lockRecord
2. 如果未重入，检查 lockRecord 是否被其他线程占用
   1. 如果被其他线程占用，则自旋等待，自旋超限后升级重量锁
   2. 如果未被其他线程占用，则取出 lockRecord 中存的指针地址，然后再用自己的 markword 做 CAS 替换

![image.png](https://cdn.nlark.com/yuque/0/2022/png/28316065/1667378600788-2593c748-5642-4a73-b8e2-06c3256d91fe.png#averageHue=%23202020&clientId=ucddd8009-ede9-4&from=paste&id=u657928fc&name=image.png&originHeight=908&originWidth=920&originalType=url&ratio=1&rotation=0&showTitle=false&size=81252&status=done&style=none&taskId=ua9eb288b-ff4a-4b02-9a9f-dac06ddf5f6&title=)
### 解锁过程
![image.png](https://cdn.nlark.com/yuque/0/2022/png/28316065/1667378668087-133f5b72-a329-4025-920b-7460991dae2f.png#averageHue=%23f8f8f8&clientId=ucddd8009-ede9-4&from=paste&id=ufda0acb1&name=image.png&originHeight=689&originWidth=778&originalType=url&ratio=1&rotation=0&showTitle=false&size=53459&status=done&style=none&taskId=u08f45442-2849-43dd-910c-1d5da043b3f&title=)
## 自适应的自旋
在JDK 6中对自旋锁的优化，引入了自适应的自旋。
自适应意味着自旋的时间不再是固定的了，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定的。如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也很有可能再次成功，进而允许自旋等待持续相对更长的时间，比如持续100次忙循环。
另一方面，如果对于某个锁，自旋很少成功获得过锁，那在以后要获取这个锁时将有可能直接省略掉自旋过程，以避免浪费处理器资源。有了自适应自旋，随着程序运行时间的增长及性能监控信息的不断完善，虚拟机对程序锁的状况预测就会越来越精准，虚拟机就会变得越来越“聪明”了
## 重量级锁
每个对象会有一个 objectMonitor 的 C++ 对象生成， 通过地址指向对方，后面的逻辑都是通过 C++来实现。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/28316065/1667379948987-9a09f3fe-cc69-4c7b-b784-d1e97299a60d.png#averageHue=%23f4f4f4&clientId=ucddd8009-ede9-4&from=paste&id=u253aab48&name=image.png&originHeight=148&originWidth=1047&originalType=url&ratio=1&rotation=0&showTitle=false&size=8178&status=done&style=none&taskId=u42e24dca-b163-4bef-9934-ed8abd02d15&title=)
### 升级为重量级锁的条件

1. 轻量级 -> 重量级，达到自适应自旋次数
2. 无锁/偏向锁 -> 重量级，**调用 Object.wait() 方法，则会直接升级为重量级锁**
### markword
对象头中的markwod，和轻量级锁中的处理类似， 被存入了objectMonitor对象的header字段中了。
