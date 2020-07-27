## List

### ArrayList源码

动态数组 初始大小为10 最大为Integer的最大容量

```java
transient Object[] elementData; // non-private to simplify nested class access
```

- add方法 检查是否扩容，通过size给数组set值，故为o（1）的时间复杂度 如果是插入 则调用System.ArrayCopy方法实现数组赋值
- grow方法 当前大小 * 1.5 使用System.ArrayCopy方法实现数组赋值 
- get方法  通过数组获得值
- Iter和ListIter 拥有两个迭代器 后者继承前者 可实现双向访问的功能
- remove方法 通过System.ArrayCopy方法覆盖掉这个值


### LinkedList源码

私有类Node结点 包含前置和后置指针，具体值

```java
//记录头节点和尾节点
transient Node<E> first;
transient Node<E> last;
```

- add方法 直接在尾节点后新增一个节点 如果是插入 则遍历到节点的位置 在其前方插入
- get方法 比较index在链表前半部分还是后半部分 前半部分则前序遍历 否则后续遍历

### CopyOnWriteArrayList源码

实现线程安全 具体方法是在写操作(add,set,remove)方法时上锁
并且使用volatile修饰动态数组 以保证其可见性

```java
final transient ReentrantLock lock = new ReentrantLock();
private transient volatile Object[] array;
```

## Set

### HashSet源码

内部调用的时hashMap的方法

## Queue

### ArrayDeque源码

与ArrayList实现原理基本相同

### ArrayBlockingQueue源码

为实现阻塞take()与put(E e)方法使用 锁与等待唤醒机制

```java
final Object[] items;
int takeIndex;
int putIndex;
final ReentrantLock lock;
private final Condition notEmpty;
private final Condition notFull;
```

- put方法 调用方法时上锁 并判断数组是否填满 如果填满 则通过Condition阻塞 如果没填满 则入队 并唤醒出队线程
- take方法 调用方法时上锁 并判断数组是否为空 如果为空 则通过Condition阻塞 如果不为空 则出队 并唤醒入队线程

### PriorityQueue源码

优先级队列,通过一个比较器来比较队列中的值 初始大小为11 实现原理为最小堆

```java
private final Comparator<? super E> comparator;
```

- grow方法 如果size小于64 size*2+2 如果size大于64 size*1.5
- offer方法 根据最小堆的特征父节点i的左子树为2*i+1，右子树为2*i+2 由此通过冒泡的方法确定新增元素的位置
- poll方法 去掉最小的节点后重构最小堆 原理同上

```java
private void siftUpUsingComparator(int k, E x) {
        while (k > 0) {
            int parent = (k - 1) >>> 1;
            Object e = queue[parent];
            if (comparator.compare(x, (E) e) >= 0)
                break;
            queue[k] = e;
            k = parent;
        }
        queue[k] = x;
    }
```

### DelayQueue

内部也是根据优先级队列实现的，比较器的比较规则是time小的FutureTask在前，如果time相同，则比较是哪个先提交的任务


## map

### HashMap源码

使用node内部类 同样是数组加链表

- get方法 
    - 计算hash值 
    - 判断数组为空或数组大小为0 
    - 判断第一个hash到的第一个位置是否有值
    - 判断为TreeNode还是Node
- put方法 
    - 判断是否初始化数组 
    - 计算hash值
    - 判断第一个hash到的第一个位置是否有值
    - 判断为TreeNode还是Node 遍历
    - 根据onlyIfAbsent参数返回原有值
- resize 新建一个新数组 将旧数组的值 并放入新数组中 这里会将hash值&容器长度，来判断哪些是高位的链表，哪些是低位的链表，这样在
赋值给新数组时，可以将不同桶的链表赋值区分开

- remove 计算hash值 找到该值所在的位置 是第一个 还是链表，红黑树 ，根据不同位置删除方法不同


### ConcurrentHashMap

同样是Node内部类 只不过value和next引用被标记为volatile

- get方法
    - 计算hash值
    - 判断第一个位置是否有值
    - 判断是否正在扩容，如果在扩容 通过ForwardingNode的find方法找到该节点
    - 遍历得到此节点
    
- put方法
    - 判断是否初始化数组 
    - 判断第一个位置是否为空，为空则通过cas插入该数据
    - 判断map是否正在在扩容，如果在扩容就加入扩容的操作 也就是并发扩容 大概是有一个ForwardingNode来记录被遍历过的节点
    - 对要遍历的坑位加锁 并判断为树还是链表
    - 判断是否要转为红黑树 这里注意map大小小于64 就不必转为红黑树 而是进行扩容
    - 判断是否要扩容
   
- initTale方法
    判断数组可用大小是否小于0 如果小于0线程就转入就绪状态，如果不小于0，就将可用大小设为-1，并为其初始化Node数组    






