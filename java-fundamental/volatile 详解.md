# 一个案例
```java
public class Test {

  public static void main(String[] args) {
    final TestSign testSign = new TestSign();

    Thread thread01 = new Thread(testSign);
    Thread thread02 =
        new Thread(
            () -> {
              try {
                Thread.sleep(3000);
              } catch (InterruptedException ignore) {
              }
              testSign.sign = true;
              System.out.println("vt.sign = true 通知 while (!sign) 结束！");
            });

    thread01.start();
    thread02.start();
  }
}

class TestSign implements Runnable {

  public boolean sign = false;

  public void run() {
    while (!sign) {}
    System.out.println("test");
  }
}
```
这段代码，希望 thread02 线程在运行时通过改变 sign 的值来让 thread01 线程的死循环被打破，输出 test
运行之后发现代码并不会输出 test，而是一直处于死循环
而将 sign 属性**加上 volatile 关键字**之后，会输出 test，表示死循环被打破了
volatile 关键字是 JVM 提供的轻量级的同步机制，用来修饰变量，用来保证变量对所有线程的可见性
所以这个属性值在被修改之后，其他的线程能及时发现它的变化
# volatile 是如何保证可见性的
## 无 volatile
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1679838299286-486cd9de-f908-4fc4-be89-ce2b7c6bcf93.png#averageHue=%23efefef&clientId=ue77fcc04-7b5d-4&from=paste&height=341&id=uf9e8c05c&name=image.png&originHeight=852&originWidth=1858&originalType=binary&ratio=2.5&rotation=0&showTitle=false&size=138766&status=done&style=none&taskId=ucbeb3f6f-2a5f-4a51-bc47-73ef2535ae6&title=&width=743.2)
当 sign 的值发生改变时，如果没有 volatile 修饰时，线程 01 对变量进行操作，线程 02 并不会感知到
## 有 volatile
![image.png](https://cdn.nlark.com/yuque/0/2023/png/28316065/1679839105121-c2f40709-8fb9-40b5-a851-eb76599967f3.png#averageHue=%23efeeee&clientId=ue77fcc04-7b5d-4&from=paste&height=294&id=ua9ac0517&name=image.png&originHeight=734&originWidth=1868&originalType=binary&ratio=2.5&rotation=0&showTitle=false&size=141874&status=done&style=none&taskId=u102e39c2-fe9d-469a-8585-62cc1a04a02&title=&width=747.2)
当用 volatile 修饰时，线程 01 对变量进行修改，会把变量的值强制刷新到主内存中，当线程 02 获取 sign 的值时，会强制将 CPU 高速缓存中的值过期掉，去主内存拿最新的值
可以通过深入查看编译之后汇编指令中，有 volatile 关键字和没有 volatile 关键字的区别在于多了一个 lock 的前缀指令，它其实相当于一个内存屏障，保证以下三点：

1. 将本处理器的缓存写入内存
2. 重排序的时候不能把后面的指令重排序到内存屏障之前的位置
3. 如果是写入的操作会导致其他处理器中对应的内存无效

这里的1、3就保证了变量内存的可见性
# 一个觉得奇怪的现象
现在将 TestSign 类中的 while 循环中加上 1 秒休眠，而不使用 volatile 修饰
```java
class TestSign implements Runnable {

  public boolean sign = false;

  public void run() {
    while (!sign) {
      try {
        TimeUnit.SECONDS.sleep(1);
      } catch (InterruptedException e) {
        throw new RuntimeException(e);
      }    }
    System.out.println("test");
  }
}
```
此时再次运行 main，会发现程序能正常运行，这是为什么？
原因在于 CPU 是比较勤奋的，如果发现自己有空闲的时间，会尝试去主内存刷新自己的缓存
# volatile 保证有序性的例子
双检单例模式的代码如下
```java
public class Singleton {

  private static volatile Singleton singleton;

  private Singleton() {}

  public static Singleton getSingleton() {
    if (singleton == null) {
      synchronized (Singleton.class) {
        if (singleton == null) {
          singleton = new Singleton();
        }
      }
    }
    return singleton;
  }
}
```
new 一个对象可以分为三步：

1. 给对象分配内存
2. 初始化对象
3. 返回对象在堆上的引用

当没有 volatile 去修饰变量的时候，创建对象并不一定会按照 123 的顺序去执行
假设线程 A 完成了 13，尚未完成 2，此时线程 B 去尝试检查是否为 null，此时发现已经有地址，此时去使用变量的时候就会出现问题，此时就需要 volatile 去禁止指令重排序来保证一定的有序

