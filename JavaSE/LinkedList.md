## LinkedList

双向链表结构，保证插入和删除的效率，读取比较慢

静态内部类Node，一个前驱指针next，一个后驱指针prev

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

维护头尾指针

```java
transient Node<E> first;
transient Node<E> last;
```

读取数据，根据索引距头指针和尾指针的距离，来选择不同的方向

```java
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
Node<E> node(int index) {
    // assert isElementIndex(index);
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

