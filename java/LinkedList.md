

# LinkedList

LinkedList的作者都说自己从来不用LinkedList。从使用场景来看，需要LinkedList的地方，ArrayList也都能做到。



## 存储容器

```java
transient int size = 0;


transient Node<E> first;


transient Node<E> last;

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

容器为双向链表，LinkedList维护了链表的头指针和尾指针。



## 构造方法

```java
public LinkedList() {
}


public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```

构造方法很简单。



## add

```java
public boolean add(E e) {
    linkLast(e);
    return true;
}

void linkLast(E e) {
    //链表尾节点
    final Node<E> l = last;
    //新建Node
    final Node<E> newNode = new Node<>(l, e, null);
    //尾节点指向新建的节点
    last = newNode;
    //如果之前没有尾节点，说明链表无节点，该节点也是头节点
    if (l == null)
        first = newNode;
    else
        //前尾节点的next指向该节点
        l.next = newNode;
    size++;
    modCount++;
}

public void add(int index, E element) {
    //判断是否越界
    checkPositionIndex(index);

    //如果添加在最后
    if (index == size)
        linkLast(element);
    else
        //把元素添加到改下标节点之前
        linkBefore(element, node(index));
}

private boolean isPositionIndex(int index) {
    return index >= 0 && index <= size;
}

//根据下标找节点
Node<E> node(int index) {
    // assert isElementIndex(index);
	
    //如果在左半边，从first开始找
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        //如果在右半边，从last开始找
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}

void linkBefore(E e, Node<E> succ) {
	//获取指定节点的前驱节点
    final Node<E> pred = succ.prev;
    //创建新节点
    final Node<E> newNode = new Node<>(pred, e, succ);
    //前驱的next指向新节点，新节点的next指向指定节点
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```





## remove

```java
public E remove(int index) {
    //检查越界
    checkElementIndex(index);
    //找到指定下标节点后移除
    return unlink(node(index));
}


//移除指定节点，前驱节点的next指向该节点的next，该节点的后续节点的prev指向该节点的前驱节点
E unlink(Node<E> x) {
    // assert x != null;
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}
```



## get

```java
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}

//根据下标找节点
Node<E> node(int index) {
    // assert isElementIndex(index);
	
    //如果在左半边，从first开始找
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        //如果在右半边，从last开始找
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

可以看到，链表不像数组可以随机访问，需要遍历链表来确定具体的位置。



## set

```java
public E set(int index, E element) {
    checkElementIndex(index);
    Node<E> x = node(index);
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}

//根据下标找节点
Node<E> node(int index) {
    // assert isElementIndex(index);
	
    //如果在左半边，从first开始找
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        //如果在右半边，从last开始找
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

set对null值没有判断，可以存储null值。



## 总结

LinkedList使用链表做存储，相比于数组，无需连续的空间，且没有扩缩容的操作。



时间复杂度

+ 查询：需要遍历链表，O(n)
+ 插入：对插入本身来说，只需要修改前后指针O(1)，但需要先找到对应的节点，O(n)
+ 删除：对于删除本身，不需要像数组那样做数据的移动，O(1)，但需要先找到对应的节点，O(n)
