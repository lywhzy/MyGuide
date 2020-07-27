## 队列同步器源码

同步器会维护一个FIFO同步队列，有一个头节点引用和尾节点引用指向这个队列，未获得同步状态的线程入队开始自旋

每一个请求访问失败的线程都被封装成Node节点，如果当前线程获取同步状态失败，则加入到同步队列中

```java
        //Node节点属性
        volatile int waitStatus; //当前线程的等待状态
        volatile Node prev;//前驱线程
        volatile Node next;//后驱线程
        volatile Thread thread;//当前线程的引用
        Node nextWaiter;//等待队列的后继节点
```

- acquire方法中的入队过程
    - 首先通过cas进行一次快速入队，将Node插入队列后方，变成尾节点
    - 如果cas失败，就死循环一直cas直到成功
- acquire方法中的自旋过程
    - 线程入队后一直死循环直到该节点前驱节点为头节点且获取同步状态成功
    - 将该节点设为头节点 并退出循环
- release方法中通过LockSupport来唤醒头节点的后继节点
- acquireShared方法 同acquire

## ReentrantLock

内部实现了两个队列同步器 分别对应公平锁和非公平锁

- 可重入 在tryAcquire中如果state为0则通过cas更改state并设置当前线程，如果不为0，则判断是否为当前线程，是则+1 否则返回false

- 公平锁 增加在tryAcquire中获取同步状态的条件 判断当前是否有节点正在等待，如果有，就不能获取等待状态

- 等待可中断 因为AQS才得以实现

## ReentrantReadWriteLock

内部也是队列同步器 state被拆分为高16位表示读锁，低16位表示写锁

- tryAcquire 在ReentrantLock的基础上加上了当前是否获取过读锁的判断，如果已经获取过读锁，那么就不会获取写锁

- tryAcquireShared 在ReentrantLock的基础上加上了 是否有其它的线程获取写锁，如果有，就不能获取等待状态

## Condition

作为AQS的内部类 维护一个FIFO的等待队列 也是有Node连接的 通过NextWaiter属性

- await方法 可以理解为将head指向的头节点阻塞，并将其加入到等待队列中，然后唤醒同步队列中的下一个节点

- signal方法 可以理解为将等待队列中第一个节点加入到同步队列中 然后使用LockSupport唤醒它

## ThreadLocal

这里要说的是其与Thread的关系，在Thread中拥有一个key为ThreadLocal，value为值的map，用来记录线程的本地变量，那么ThreadLocal创建出来
的目的就是要把自己注册到当前线程，然后从Thread中获取map并进行修改

### ThreadLocal引发内存泄漏的原因

- Thread一直在运行 则持有ThreadLocalMap的引用
- ThreadLocal引用被设置为null 则ThreadLocal实例当前只通过一个软引用 引用到ThreadLocalMap的Entry上的key
- 触发gc，使得ThreadLocal实例被回收，但是由于ThreadLocalMap依然存在，所以Entry实例上的value并没有被gc 这样就造成了内存泄漏

## ScheduledThreadPoolExecutor

任务队列为DelayQueue

