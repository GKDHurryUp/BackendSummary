# ConcurrentHashMap #



## 为什么不使用HashTable？ ##
1. HashTable通过在put、get方法上加synchronized来锁多个线程同一个hashmap对象，也就是同一把锁
2. 而在多个线程中，调用put方法，他们算出来的数组索引值可能是不同的，要插入的位置不一样，但是依旧synchronized锁了整个对象，效率就有些低了

## ConcurrentHashMap锁的基本思想 ##
如果数组每个位置加一把锁，根据算出来的索引值来判断这个锁是否被占用，效率是不是就可以提升了？---分段锁

每个线程有自己的内存，如何让线程操作机器的内存呢？

Unsafe工具，有compareAndSwapObject，compareAndSwapInt(修改的对象，成员变量的OFFSET)方法   ---CAS算法

ConcurrentHashMap中的类加载器为Bootstrap类加载器，只有这种类可以使用getUnsafe()方法，否则返回null(如自己定义的Person类)

## JDK1.7下的ConcurrentHashMap ##
1. 底层采用分段数组+链表来实现
2. 使用分段锁对整个通数组进行了分段分割（Segment），每个锁只锁容器的一部分，多线程访问容器里不同数据段的数据，就不会存在锁竞争，提高并发访问效率
3. Segment作为数组，继承了ReentrantLock，所以Segment是一种可重入锁，扮演锁的角色。（可重入就是说某个线程已经获得某个锁，可以再次获取锁而不会出现死锁）
		static final class Segment<K,V> extends ReentrantLock

ConcurrentHashMap
	Segment[]
Segment	
	HashEntry[]

HashEntry
	key 
	val
	hash

## get方法需要加锁吗？为什么？ ##
1. get不需要加锁，它将要使用的共享变量都定义成volatile类型，保证多线程读取一致。
2. 统计当前Segement大小的count字段和用于存储值的HashEntry的value

## 为什么定位Segment的散列算法和定位HashEntry相与的值不一样？ ##
定位Segment使用的是元素的hashcode通过再散列后得到的值的高位，而定位HashEntry直接使用的是再散列后的值。其目的是避免两次散列后的值一样
第一次使用hash的高位确定在Segment[]索引，也是&运算
第二次使用hash的低位确定在HashEntry[]索引，也是&运算

numberOfLeadingZeros() 返回高位0的个数	

## put方法 ##
（1）是否需要扩容
是否需要扩容在插入元素前会先判断Segment里的HashEntry数组是否超过容量
（2）如何扩容
1. 首先会创建一个容量是原来容量两倍的数组，然后将原数组里的元素进行再散列后插入到新的数组。
2. ConcurrentHashMap不会对整个容器进行扩容，而只对某个segment进行扩容

## size方法 ##
1. Segment里的全局变量count是volatile，相加虽然相加时可以获取每个Segment的count的最新值，但是可能累加前使用的count发生了变化，那么统计结果就不准了。(volatile++操作不保证原子性)
2. 最安全做法：统计size时，把所有put、remove、clear方法锁住
3. ConcurrentHashMap的做法是先尝试2次通过不锁住Segment的方式来统计各个Segment大小，如果统计的过程中，容器的count发生了变化，则再采用加锁的方式来统计所有Segment的大小。

---------

## JDK1.8下的ConcurrentHashMap ##

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {

    transient volatile Node<K,V>[] table;	//默认为null，初始化发生在第一次插入操作，默认大小为16的数组，用来存储Node节点数据，扩容时大小总是2的幂次方。
    private transient volatile Node<K,V>[] nextTable;	//扩容时新生成的数组
    private transient volatile int sizeCtl;		//控制table的初始化和扩容操作
	//... 省略部分代码
}
```

Node结构，保存key，value及key的hash值的数据结构

```java
class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;
    //... 省略部分代码
}
```

1. 数据结构与HashMap一样，直接使用Node数组+链表+红黑树的数据结构来实现
2. 取消了Segment分段锁（ReentrantLock），并发控制使用synchronized和CAS来操作。
3. synchronized只锁定当前链表或者红黑树二叉树的首节点（降低锁的粒度），只要hash不冲突，就不会产生并发。

