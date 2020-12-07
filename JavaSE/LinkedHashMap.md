# LinkedHashMap #


LinkedHashMap是HashMap的子类，它通过维护一个额外的**双向链表**保证了迭代的顺序（插入顺序和访问顺序，默认插入）![](https://img-blog.csdn.net/20170512160734275?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvanVzdGxvdmV5b3Vf/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

HashMap与LinkedHashMap的Entry结构示意图如下图所示：
![](https://img-blog.csdn.net/20170512155609530?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvanVzdGxvdmV5b3Vf/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## LinkedHashMap 在 JDK 中的定义 ##
1. 类结构定义
	```java
	public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V> {
		...
	}
	```
	
2. 成员变量定义

	```java
	transient LinkedHashMap.Entry<K,V> head; // 双向链表的表头元素
	
	transient LinkedHashMap.Entry<K,V> tail; // 双向链表的表尾元素
	
	final boolean accessOrder; //true表示按照访问顺序迭代，false时表示按照插入顺序（默认） 
	```
3. 基本元素Entry

	```java
   static class Entry<K,V> extends HashMap.Node<K,V> {
       Entry<K,V> before, after;
       Entry(int hash, K key, V value, Node<K,V> next) {
           super(hash, key, value, next);
       }
    }  
   ```

4. HashMap中的Node

   ```java
   static class Node<K,V> implements Map.Entry<K,V> {
       final int hash;
       final K key;
       V value;
       Node<K,V> next;
       ...
   }
   ```

   

##  LinkedHashMap 与 LRU(Least recently used，最近最少使用)算法 ##
- 当accessOrder标志位为true时，表示双向链表中的元素按照访问的先后顺序排列，==get方法和put方法==都会调用==recordAccess()==方法（在HashMap中为空实现，它提供LRU实现）使得最近使用的**Entry移到双向链表的末尾**；

- 当accessOrder标志位为false时，表示双向链表中的元素按照Entry插入LinkedHashMap到中的先后顺序排序，即每次put到LinkedHashMap中的Entry都放在双向链表的尾部，这样遍历双向链表时，Entry的输出顺序便和插入的顺序一致

## LinkedHashMap 有序性原理分析 ##
LinkedHashMap重写了HashMap 的迭代器，它使用其维护的双向链表进行迭代输出。定义了一个抽象类LinkedHashIterator，LinkedKeyIterator，LinkedValueIterator，LinkedEntryIterator继承该抽象类，实现迭代

```java
abstract class LinkedHashIterator {
    LinkedHashMap.Entry<K,V> next;
    LinkedHashMap.Entry<K,V> current;
    int expectedModCount;

    LinkedHashIterator() {
        next = head;
        expectedModCount = modCount;
        current = null;
    }
	...
｝
final class LinkedKeyIterator extends LinkedHashIterator
    implements Iterator<K> {
    public final K next() { return nextNode().getKey(); }
}

final class LinkedValueIterator extends LinkedHashIterator
    implements Iterator<V> {
    public final V next() { return nextNode().value; }
}

final class LinkedEntryIterator extends LinkedHashIterator
    implements Iterator<Map.Entry<K,V>> {
    public final Map.Entry<K,V> next() { return nextNode(); }
}
```