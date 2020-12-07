# ConcurrentLinkedQueue线程安全队列

是一个基于链接节点的无界线程安全队列，它采用先进先出的规则对节点进行排序，当我们添加一个元素的时候，它会添加到队列的尾部；当我们获取一个元素时，它会返回队列头部的元素。

## 实现一个线程安全的队列 ##
1. 阻塞算法
（1）一个锁，入队和出队用同一把锁
（2）两个锁，入队和出队用不同的锁
2. 非阻塞算法
CAS+循环

## ConcurrentLinkedQueue使用约定：

1. **不允许null入列**
2. 在入队的最后一个元素的next为null
3. 队列中所有未删除的节点的item都不能为null且都能从head节点遍历到
4. 删除节点是将item设置为null, 队列迭代时跳过item为null节点
5. **head节点跟tail不一定指向头节点或尾节点，可能存在滞后性**
6. **size方法不是常数级别**，而是O(n)遍历



## 源码 ##

~~~java
public class ConcurrentLinkedQueue<E> extends AbstractQueue<E>
    implements Queue<E>, java.io.Serializable {

    // 静态内部类 Node节点，保护当前节点的值item，和next指针
    private static class Node<E> { 
        volatile E item;
        volatile Node<E> next;
    }
    private transient volatile Node<E> head; // 头结点
    private transient volatile Node<E> tail; // 尾节点
}
~~~

1. 首先构造函数会初始化一个哑结点

	```java
	public ConcurrentLinkedQueue() {
	    head = tail = new Node<E>(null);
	}
	```
2. add/offer方法，向尾节点添加元素

	​	
	
	```java
	public boolean offer(E e) {
	    checkNotNull(e); // 检查不为null元素
	    final Node<E> newNode = new Node<E>(e); // 创建一个要入队列的Node
	    for (Node<E> t = tail, p = t;;) { // for内定义两个变量t, p都指向队尾，死循环
	        Node<E> q = p.next;  // q为取得队尾的元素的下一个
	        if (q == null) { // 如果q为空
	            // p is last node
	            if (p.casNext(null, newNode)) { //对p进行CAS操作，如果p为null，则把newNode赋值给p的next节点
	                // Successful CAS is the linearization point
	                // for e to become an element of this queue,
	                // and for newNode to become "live".
	                if (p != t) // hop two nodes at a time
	                    casTail(t, newNode);  // Failure is OK.
	                return true;
	            }
	            // Lost CAS race to another thread; re-read next
	        }
	        else if (p == q)
	            // We have fallen off list.  If tail is unchanged, it
	            // will also be off-list, in which case we need to
	            // jump to head, from which all live nodes are always
	            // reachable.  Else the new tail is a better bet.
	            p = (t != (t = tail)) ? t : head;
	        else
	            // Check for tail updates after two hops.
	            p = (p != t && t != (t = tail)) ? t : q;
	    }
	}
	```

第一次添加元素时：

3. size方法，由于队列异步的特性，没有采用时间常数的做法，而是O(n)遍历，但是在执行size方法的时候进行offer或者remove可能会使结果不准确。

		```java
	public int size() {
	    int count = 0;
	    for (Node<E> p = first(); p != null; p = succ(p))
	    if (p.item != null)
	    // Collection.size() spec says to max out
	    if (++count == Integer.MAX_VALUE)
	    break;
	    return count;
	}
	```
	
	