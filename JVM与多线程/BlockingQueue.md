## BlockingQueue

支持两个附加操作的队列

- 在队列为空，获取元素的线程会等待队列变为非空
- 当队列满时，存储元素的线程会等待队列可用

![](../imgs\BlockingQueue.png)

### Condition

- await
- signal
- signalAll

在对一个空队列进行take操作时，会取得锁权限，但发现队列是空的，需要等待有值插入才能继续，此时如果不释放锁就会造成死锁，因此要进行释放锁并且等待（await），进入一个Waitting Room，这个Waitting Room就相当于Condition。

### ArrayBlockingQueue

底层使用数组来存放，同时有一个take指针和put指针，并且使用lock配合Condition来实现等待通知

一把锁+2个条件变量

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, java.io.Serializable {
	/** The queued items */
    final Object[] items;

    /** items index for next take, poll, peek or remove */
    int takeIndex;

    /** items index for next put, offer, or add */
    int putIndex;

    /** Number of elements in the queue */
    int count;
    
    /** Main lock guarding all access */
    final ReentrantLock lock;

    /** Condition for waiting takes */
    private final Condition notEmpty;

    /** Condition for waiting puts */
    private final Condition notFull;
    ..
}
```

在put时，先加锁，通过nutFull来进行等待

```java
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        // 并不能保证在Waitting Room出来的线程一定取得锁继续执行，它还会和外部来的线程进行竞争
        while (count == items.length)
            notFull.await();
        enqueue(e); //notEmpty.signal();
    } finally {
        lock.unlock();
    }
}
```

在take时，先加锁，通过notEmpty来进行等待

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0)
            notEmpty.await();
        return dequeue(); // notFull.signal();
    } finally {
        lock.unlock();
    }
}
```

使用Synchronized关键字也可以实现这种效果，但是==Synchronized只有一个Waitting Room==，它的唤醒会把所有的等待线程都唤醒，效率低。

而lock+Condition创建**两个**Waitting Room，可以**指定唤醒**非空的Waitting Room，或者非满的Waitting Room



### LinkedBlockingQueue

低层使用链表，维护头尾指针，并配置**两把锁+2个Condition**	

```java
transient Node<E> head;

/**
 * Tail of linked list.
 * Invariant: last.next == null
 */
private transient Node<E> last;

/** Lock held by take, poll, etc */
private final ReentrantLock takeLock = new ReentrantLock();

/** Wait queue for waiting takes */
private final Condition notEmpty = takeLock.newCondition();

/** Lock held by put, offer, etc */
private final ReentrantLock putLock = new ReentrantLock();

/** Wait queue for waiting puts */
private final Condition notFull = putLock.newCondition();
```



### SynchronousQueue

同步队列，没有容量，在put时候会进行阻塞，等待匹配获取。

内部使用SNode来保存put操作存入的节点，与take操作存入的节点，他们的节点有不同的状态，Request，Data，根据这个状态进行匹配操作

```java
static final class SNode {
    volatile SNode next;        // next node in stack
    volatile SNode match;       // the node matched to this
    volatile Thread waiter;     // to control park/unpark
    Object item;                // data; or null for REQUESTs
    int mode;
    ...
}
```



用于两个线程之间高效的交换数据

### LinkedTransferQueue

额外有一个transfer方法，需要放入的数据被取走，才会返回。与SynchronousQueue不同的是，它有容量





## 高并发下队列的性能优化

### 单写原则 Single Writer Principle

- 只有一个线程对资源进行写操作，无需CPU浪费管理资源。上下文切换

### 伪共享 False Sharing

当一个CPU核写数据，而另外一个CPU读数据，但因为数据挨在一起，位于**同一个缓存行内**Cache Line。**读数据的CPU每次就需要到主存中读取数据**，导致缓存行失效，缓存效率很低

解决：

1. Padding填充

   使用子类继承父类，让编译器不会优化掉

2. @Contended

   JDK1.8

### 环形队列预分配，零GC

### 批量生产及消费

### 位运算代替求余取模运算

### Distruptor框架