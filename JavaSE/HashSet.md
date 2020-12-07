## HashSet ##
HashSet是基于HashMap来实现的，操作很简单，更像是对HashMap做了一次“封装”，而且只使用了HashMap的key来实现各种特性。

1. 属性

    ```java
    private transient HashMap<E, Object> map;
    // PRESENT则是用来造一个假的value来用的。
    private static final Object PRESENT = new Object();
    ```

2. 基本操作
	
		``` java
	public boolean add(E e) {
	    return map.put(e, PRESENT)==null;
	}
	public boolean remove(Object o) {
	    return map.remove(o)==PRESENT;
	}
	public boolean contains(Object o) {
	    return map.containsKey(o);
	}
	public int size() {
	    return map.size();
	}
	```
	

## HashSet如何去重？

HashSet低层是使用HashMap来进行实现

```java
if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
```

先比较hashCode（大部分情况不会走到后边），因为hashCode有很低几率发送碰撞，需要有后序的判断，判断这两个对象地址是否相同或者调用equals比较

在add方法中会调用HashMap的put方法，传入的对象hashCode值，然后再进行rehash，计算出元素在数组索引的位置

- 如果该位置没有元素，直接存储
- 如果有元素，会调用equals方法进行比较

