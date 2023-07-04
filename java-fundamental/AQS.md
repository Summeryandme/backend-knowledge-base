# AQS 简介
AQS (AbstractQueuedSynchronizer) 是一个用来构建锁和同步器的框架，使用 AQS 框架可以简单并且高效的构造出应用广泛的大量的同步器，例如常见的 ReentrantLock，Semaphore 等等
## 核心思想
AQS 的核心思想是：当请求的共享资源空闲，即将当前请求资源的线程设置为有效的工作线程，并且将共享资源设置为锁定状态，如果请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，这个机制 AQS 是通过 CLH 队列实现的，即将暂时获取不到锁的线程加入到队列之中
> CLH (Craig,Landin,and Hagersten) 队列是一个虚拟的双向队列（虚拟的双向队列即不存在队列实例，仅存在结点之间的关联关系）。AQS是将每条请求共享资源的线程封装成一个 CLH 锁队列的一个结点(Node)来实现锁的分配。


AQS 使用一个 int 类型的成员变量来表示同步状态，通过内置的 FIFO 队列来完成获取资源线程的排队工作，AQS 使用 CAS 对该同步状态进行原子操作实现对其值的修改
```java
//共享变量，使用volatile修饰保证线程可见性
private volatile int state;
```
## AQS 对资源的共享方式

- Exclusive（独占）：只有一个线程可以拿到共享资源，例如 ReentrantLock，更进一步的可以分为公平锁和非公平锁
- Shared（共享）：多个线程可以同时执行，例如 Semaphore，CountDownLatch，CyclicBarrier 等等

ReentrantReadWriteLock 可以看成是组合式，因为 ReentrantReadWriteLock 也就是读写锁允许多个线程同时对某一资源进行读。不同的自定义同步器争用共享资源的方式也不同。自定义同步器在实现时**只需要实现共享资源 state 的获取与释放方式**即可，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS 已经在上层已经帮我们实现好了。
## AQS 的模版模式
> 同步器的设计是基于模板方法模式的，如果需要自定义同步器一般的方式是这样(模板方法模式很经典的一个应用)：

使用者继承 AbstractQueuedSynchronizer 并重写指定的方法（一般就是对于 state 的获取和释放），这和普通的实现接口有点差异，自定义同步器需要重写以下方法：
```java
isHeldExclusively()//该线程是否正在独占资源。只有用到 condition 才需要去实现它。
tryAcquire(int)//独占方式。尝试获取资源，成功则返回true，失败则返回false。
tryRelease(int)//独占方式。尝试释放资源，成功则返回true，失败则返回false。
tryAcquireShared(int)//共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
tryReleaseShared(int)//共享方式。尝试释放资源，成功则返回true，失败则返回false。
```
默认情况下，每个方法都会抛出`UnsupportedOperationException`异常，这些方法的实现必须要是内部线程安全的，通常应尽量简短，不要阻塞，AQS 类中的其他方法都是 final 类型的，只有这几个方法可以被其他类所使用

以 ReentrantLock 为例，state 初始化为 0，表示未锁定状态，A 线程调用 lock() 时，会调用 tryAquire() 独占该锁并将 state + 1，此后，其他线程再 tryAquire() 的时候就会失败，直到 A 线程 unlock() 到 state=0 为止，其他线程才有机会获得锁
# AQS 数据结构
AQS 将每条请求共享资源封装成一个 CLH 锁队列的节点（Node）来实现锁的分配，其中 Sync Queue，即同步队列，是双向链表，包括 head 节点和 tail 节点，head 节点主要用于后续的调度，而 Condition Queue 不是必须的，其是一个单向链表，只有当使用 Condition 时，才会存在此单向链表，并且可能存在多个 Condition Queue
# 核心方法 - acquire
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

- 首先调用`tryAcquire`方法，调用此方法的线程会试图在独占模式下获取对象状态。此方法应该查询是否允许它在独占模式下获取对象状态，如果允许，则获取它。在 AbstractQueuedSynchronizer 源码中默认会抛出一个异常，即需要子类去重写此方法完成自己的逻辑。
- 若`tryAcquire`失败，则调用`addWaiter`方法，`addWaiter`方法完成的功能是将调用此方法的线程封装成为一个结点并放入 Sync Queue。
- 调用`acquireQueued`方法，此方法完成的功能是 Sync Queue 中的结点不断尝试获取资源，若成功，则返回 true，否则，返回 false。
## addwaiter
```java
private Node addWaiter(Node mode) {
    // 新生成一个结点，默认为独占模式
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    // 保存尾结点
    Node pred = tail;
    if (pred != null) { // 尾结点不为空，即已经被初始化
        // 将node结点的prev域连接到尾结点
        node.prev = pred; 
        if (compareAndSetTail(pred, node)) { // 比较pred是否为尾结点，是则将尾结点设置为node 
            // 设置尾结点的next域为node
            pred.next = node;
            return node; // 返回新生成的结点
        }
    }
    enq(node); // 尾结点为空(即还没有被初始化过)，或者是compareAndSetTail操作失败，则入队列
    return node;
}
```
`addWaiter` 方法使用快速添加的方式往 sync queue 尾部添加结点，如果 sync queue 队列还没有初始化，则会使用 enq 方法插入队列中
```java
private Node enq(final Node node) {
    for (;;) { // 无限循环，确保结点能够成功入队列
        // 保存尾结点
        Node t = tail;
        if (t == null) { // 尾结点为空，即还没被初始化
            if (compareAndSetHead(new Node())) // 头节点为空，并设置头节点为新生成的结点
                tail = head; // 头节点与尾结点都指向同一个新生结点
        } else { // 尾结点不为空，即已经被初始化过
            // 将node结点的prev域连接到尾结点
            node.prev = t; 
            if (compareAndSetTail(t, node)) { // 比较结点t是否为尾结点，若是则将尾结点设置为node
                // 设置尾结点的next域为node
                t.next = node; 
                return t; // 返回尾结点
            }
        }
    }
}
```
## acquireQueued
```java
// sync队列中的结点在独占且忽略中断的模式下获取(资源)
final boolean acquireQueued(final Node node, int arg) {
    // 标志
    boolean failed = true;
    try {
        // 中断标志
        boolean interrupted = false;
        for (;;) { // 无限循环
            // 获取node节点的前驱结点
            final Node p = node.predecessor(); 
            if (p == head && tryAcquire(arg)) { // 前驱为头节点并且成功获得锁
                setHead(node); // 设置头节点
                p.next = null; // help GC
                failed = false; // 设置标志
                return interrupted; 
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
# AQS 总结

- 每一个结点都是由前一个结点唤醒
- 当结点发现前驱结点是 head 并且尝试获取成功，则会轮到该线程进行
- condition queue 中的结点向 sync queue 中转移是通过 signal 操作完成的
- 当结点的状态为 SIGNAL 时，表示后面的结点需要进行
