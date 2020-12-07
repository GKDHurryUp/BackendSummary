# TreeMap

- TreeMap存储K-V键值对，通过红黑树（R-B tree）实现
- TreeMap继承了NavigableMap接口，NavigableMap接口继承了SortedMap接口，可支持一系列的导航定位以及导航操作的方法
- TreeMap因为是通过红黑树实现，红黑树结构天然支持排序，默认情况下通过Key值的自然顺序进行排序

```java
public class TreeMap<K,V>
    extends AbstractMap<K,V>
    implements NavigableMap<K,V>, Cloneable, java.io.Serializable
{
    private final Comparator<? super K> comparator;
	private transient Entry<K,V> root;
    ...
        
}
```

Entry类

```java
static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left;
    Entry<K,V> right;
    Entry<K,V> parent;
    boolean color = BLACK;
    
     public boolean equals(Object o) {
         if (!(o instanceof Map.Entry))
             return false;
         Map.Entry<?,?> e = (Map.Entry<?,?>)o;

         return valEquals(key,e.getKey()) && valEquals(value,e.getValue());
     }
    
}
```



## TreeMap如何去重？

在TreeMap中的构造函数中，可以传入对应的比较器，如果没有传入，比较器为null。它在put时候，会进行判断

- 如果说comparator为null，调用默认的compare方法按字典序进行比较
- 如果说comparator不为null，comparator的compare方法进行比较

结果小的放入左子树，大的放入右子树，结果相等的就直接返回

