# HashMap

HashMap类主要用来处理具有键值对特征的数据。HashMap是基于哈希表对Map接口的实现，HashMap具有较快的访问速度，但是遍历顺序却是不一定的，允许null值和null键。不安全

初始容量，默认16

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
```

**loadFactor**负载因子，默认0.75

```java
final float loadFactor;
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```

**threshold**，扩容阈值，表示所能容纳的键值对的临界值，默认16*0.75=12

```java
int threshold;
threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
```

**size**是HashMap中实际存在的键值对数量

```java
transient int size;
```

**modCount**字段是用来记录集合内部结构发生变化的次数，**用来快速并发报错**

```java
transient int modCount;
```

## JDK1.7

底层使用了数组+链表实现，使用一个Entry数组，存放key、value、hash值、next指针

```java
static class Entry<K,V> implements Map.Entry<K,V> {
    final K key;
    V value;
    Entry<K,V> next;
    int hash;
    ...
}
```

### 1. 如何计算索引位置？

需要根据hash值来计算数组索引下标，最简单的一个方法就是使用hash值来对数组长度求余，但是这种方法效率过慢，而使用h & (length-1) 位运算&来计算索引时，效率会高很多。

```javascript
static int indexFor(int h, int length) {
    // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
    return h & (length-1);
}
```

### 2. 为什么在HashMap中的数组大小一定要为2的幂？ ###

因为要使用数组的长度-1来计算索引位置，而此时若数组的长度为2的幂次方(例如16)，其减一的值(15)，低位全为1(0000 1111)，此时进行&运算，结果就是hash值的低位(低四位)。而若此时数组的长度不为2的幂次方，其减一的值与hash值进行&运算，就不能保证计算得到的数组下标会平均分配到0-数组长度-1，导致某几个存在数组中的链表过长。

### 3. 为什么要对hashCode()再次操作？

因为在计算索引位置时，用到了Hash值的与运算，而数组长度一般高位都为0，这时候直接进行与运算，会使得Hash值的高位失效，**只有真正的低几位参与到了索引的计算中**，会有很大概率发生**哈希碰撞**，使得数组不均分，某几个链表过长，影响效率。

使用位移操作可以让Hash值全部参与到索引的计算中来，尽可能的保证数组中的链表长度均匀

```java
final int hash(Object k) {
    int h = hashSeed;
    if (0 != h && k instanceof String) {
        return sun.misc.Hashing.stringHash32((String) k);
    }
    h ^= k.hashCode();

    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

### 4. 解决hash冲突时，为什么用链地址法，不用开发地址法？

如果index索引与当前的index有冲突，即把当前的索引index+1。如果在index+1已经存在占位现象（index+1的内容不为NULL）试图接着index+2执行，直到找到索引为内容为NULL的为止。

1. 假设有一批数据，他们计算出的索引很相似，此时就一一次次的向后查找，还需要不断的对数组进行扩容，效率很低

2. 另一个致命的问题就是在数据删除后，如果再次查询可能无法定到下一个连续数字，只能在被删结点上做删除标记，而不能真正删除结点。

### 5. 为什么使用头插法？ ###

因为方便，快速，只需要取出数组上的链表，把新的链表节点放入数组中，把新的next节点指向原来的节点，即可完成插入。

而如果使用尾插法，需要遍历到数组的尾部，效率不如头插法

```java
void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```

### 6. 为什么HashMap会形成环状链表？ ###

在resize过程中，再次重新开辟一个新的数组，若在此时，有两个线程同时进入

```java
for (Entry<K,V> e : table) {
    while(null != e) {
        Entry<K,V> next = e.next; //线程2在此阻塞
        int i = indexFor(e.hash, newCapacity);
        e.next = newTable[i];
        newTable[i] = e;
        e = next;
     }
 }
```

==**头插法，链表的顺序会颠倒**==，若一个线程在取得了初始链表的信息，然后发生了阻塞，此时这个线程内保留e和next指向的位置。另一个线程开始运行，它进行了头插法复制到新数组中，链表的顺序会发生颠倒，而被阻塞的线程此时开始工作，实际上它保存的e和next所指向的链表已经发生了颠倒，它再去进行新的复制操作，变会出现循环链表

### 7. 为什要扩容？如何进行扩容？

如果数组长度小的话，链表会越来越长，相应的get方法遍历链表的话，效率会越来越低。

==先扩容，后添加==。当hashmap中的元素个数超过**threshold**，也就是**数组大小*loadFactor**时，就会进行数组扩容，重新开辟一个**2倍大小**的数组，**重新计算每个元素在数组中的位置（1.8进行了优化）**，如果先插入扩容需要多计算一次哈希值，因此，先扩容，后添加

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    if ((size >= threshold) && (null != table[bucketIndex])) {
        resize(2 * table.length);
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }

    createEntry(hash, key, value, bucketIndex);
}
```

### 8. 为什设置负载因子为0.75？ ###

这是一个权衡的过程，如果设置为1，可以最大化利用空间，但是还是无法避免hash碰撞，链表不一定会均分在数组上，但此时get效率会下降。如果设置为0.5，hash碰撞的概率会降低，查询效率很高，但是空间需求很大。随着数组的长度的增加，hash碰撞会降低，

 理想情况下，在随机hashcode下，bin中节点的频率遵循Poisson分布这个过程（概率）是服从泊松分布，假定一个bin空和非空的概率为0.5，根据二项式定理，桶为空的概率为在取极限情况下趋于log(2)≈0.693，合理值约为0.7。

而在0.75也就是3/4和数组的容量乘积出来是整数，因此就选用0.75

### 9. 为什么常使用String做为key？

1. String是一个final修饰的类，底层是一个 final 修饰的 char 类型的数组，是immutable的，不会被继承修改
2. 用自定义对象做key的话，需要重写hashCode和equals，条件比较苛刻

## JDK1.8

HashMap采用了==数组+链表+红黑树==的存储结构。HashMap数组部分称为哈希桶，当链表长度大于等于8，链表数据将以红黑树的形式进行存储，当长度降到6时，转为链表。链表的时间复杂度为O(n)，红黑树的时间复杂度为O(log n)

每个Node结点存储着用来定位数据索引位置的hash值，k键，V值，链表下一个结点Node<K,V>;

// Map.Entry是Map的一个内部接口。内部接口只有在静态时才有意义。内部接口都是静态的。

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
    ...
}
```

在1.7的基础上，多增添了链表和红黑树的转换方面的属性

树转换为链表的阈值

```java
static final int UNTREEIFY_THRESHOLD = 6;
```

链表转换为树阈值

```java
static final int TREEIFY_THRESHOLD = 8;
```

最小树化节点数量

```java
static final int MIN_TREEIFY_CAPACITY = 64;
```




### 1. 为什么hash()计算方式相较于1.7变化了？ ###

hash()稍作简化，提高计算速度，这实际上是一个速度与效率的权衡。而且因为1.8引入的红黑树，即使碰撞发生多了，也使用红黑树进行存储，而不是链表结构

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```


### 2. 为什么JDK1.8是向链表尾部插入呢？ ###

因为1.8加入了和红黑树，在插入Node时，需要**判断链表的长度是否满足树化阈值**，就需要遍历整个链表，就没必要再插入到头部，再移动链表，只需要将链表插入到尾部即可。

### 3. 如何扩容？

==先添加，后扩容==。当hashmap中的元素个数超过**threshold**，也就是**数组大小*loadFactor**时，就会进行数组扩容，重新开辟一个**2倍大小**的数组。其实，数组扩容前的索引，和扩容后的索引要么**相同**，要么**等于旧数组的长度+扩容前的索引**，JDK1.8中使用了这种特性，==避免了rehash==，就可以先插入再扩容了（这样比较方便，而且**put有可能只是覆盖数据**，不需要扩容的，所以先插入可以排除条件）

### 4. 树化阈值为什么选择数字8？

理想情况下使用随机的哈希码，容器中节点分布在hash桶中的频率遵循泊松分布，按照泊松分布的计算公式计算出了桶中元素个数和概率的对照表，可以看到链表中元素个数为8时的概率已经非常小（0.00000006），所以原作者在选择链表元素个数时选择了8，是根据概率统计而选择的

### 5. 为什么树化阈值，反树化阈值不设置成一样的呢？ ###
1. TREEIFY_THRESHOLD=8，树化阈值，认为链表长度大于8，将链表转化为红黑树
2. UNTREEIFY_THRESHOLD=6，反树化阈值，认为节点数量小于6，将红黑树转为链表
	**防止红黑树与链表频繁的变动**

### 6. 是不是链表长度大于TREEIFY_THRESHOLD就一定会转化红黑树？ ###
不是的，在转化为红黑树之前会进行一个判断，如果Node的数量是小于MIN_TREEIFY_CAPACITY，则会先进行扩容，能不转换尽量不转换，因为开始需要一定额外的开销，也是进行了一种平衡。

### 7. 为什么要选用红黑树？而不用平衡二叉树？ ###
1. JDK基于综合考虑，为了同时保证put和get的效率，红黑树插入和查询都可以满足此要求
2. 红黑树的查找、插入和删除时间复杂度都为O(logN)，额外的开销是每个节点的存储空间都稍微增加了一点，因为一个存储红黑树节点的颜色变量。插入和删除的时间要增加一个常数因子，因为要进行旋转，平均一次插入大约需要一次旋转，因此插入的时间复杂度还是O(logN),(时间复杂度的计算要省略常数)，但实际上比普通的二叉树是要慢的。**红黑树的优点是对于有序数据的操作不会慢到O(N)的时间复杂度。**
3. 平衡二叉树避免了向线性结构演化的倾向，严格保证平衡因子不会大于1，但是**它的统计特性不如红黑树**，红黑树在进行平衡保证时的**代价要更低**一些，基于综合考虑，使用了红黑树

### 8. 为什么解决了环状链表，依然有安全问题？ ###
1. 在JDK1.8中，定义了高低指针，没有进行rehash计算（根据哈希值来&旧数组的长度，讲一个链表拆为两部分，插入新链表的两个部分），提高的性能。

2. 但是依旧会发生hash碰撞，如下代码
		
	```java
if ((p = tab[i = (n - 1) & hash]) == null) // 如果没有hash碰撞则直接插入元素
		tab[i] = newNode(hash, key, value, null);
	```
	
	判断是否出现hash碰撞，假设两个线程A、B都在进行put操作，并且hash函数计算出的插入下标是相同的（当然key不相等），当线程A执行完第六行代码后由于时间片耗尽导致被挂起，而线程B得到时间片后在该下标处插入了元素，完成了正常的插入，然后线程A获得时间片，由于之前已经进行了hash碰撞的判断，所有此时不会再进行判断，而是直接进行插入，这就导致了线程B插入的数据被线程A覆盖了，从而线程不安全。


### 9. modCount是用来干什么的？ ###
 使用modCount变量，在put、remove和clean方法里操作元素前都会将变量modCount进行加1，那么在统size前后比较modCount是否发生变化，从而得知容器的大小是否发生变化

有一点版本控制的意思，可以理解成version，在特定的操作下需要对version进行检查，适用于Fail-Fast机制。
当使用iterator去遍历某集合的过程中，其他线程修改了此集合，此时会抛出ConcurrentModificationException异常。