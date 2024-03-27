

# HashMap



## 存储容器

```java
transient Node<K,V>[] table;


static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    
    final K key;

    V value;
    
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```

HashMap底层是一个Node数组。存放了key、value、hash。

默认容量没有指定，也就是说table默认为null。



## 构造方法

```java

public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    //负载因子
    this.loadFactor = loadFactor;
    //计算阈值
    this.threshold = tableSizeFor(initialCapacity);
}

//确保数组大小时2的次幂
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}

//默认负载因子为0.75
static final float DEFAULT_LOAD_FACTOR = 0.75f;

public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}


public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```



## put存放元素

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

//根据key做hash，若key为null，则位置为0
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

static final int TREEIFY_THRESHOLD = 8;

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict) {
    Node<K,V>[] tab; 
    Node<K,V> p; 
    int n, i;
    //HashMap的数组为空，先扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //n为数组长度，hash与数组长度计算出位置i
    //若数组中该位置为空，则此次放的是hashmap的第一个元素
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    //该位置已有元素
    else {
        Node<K,V> e; K k;
        //如果首元素的key、hash与放入的新元素相等，说明要更新旧元素
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //如果是树节点，调用putTreeVal方法进行处理
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //链表
            //记录链表中元素数量
            for (int binCount = 0; ; ++binCount) {
                //遍历到链表末尾，添加新元素
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    //如果链表元素数量大于等于8，转换为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //如果元素的key、hash与放入的新元素相等，说明要更新旧元素
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //如果e不为空，说明已存在hash、key相同的节点，更新旧值，返回旧值。
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    //modCount
    ++modCount;
    //如果数组大小超过阈值，扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

put方法，会先根据hash&容量，计算出位置。若位置上无节点，则该元素为头节点，若已有链表，则从链表末尾插入(尾插法)，若链表元素大于等于8个，转换成红黑树。

jdk7会采用头插法，但有死循环的可能。

## resize扩容

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; //16

final Node<K,V>[] resize() {
    //旧数组
    Node<K,V>[] oldTab = table;
    //旧大小
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    //旧阈值
    int oldThr = threshold;
    int newCap, newThr = 0;
    //如果旧数组不为空
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //旧数组不为空，新阈值为旧阈值的两倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // 如果旧容量为0但阈值大于0,鑫融联设置为旧阈值
    else if (oldThr > 0) 
        newCap = oldThr;
    else {               
        // 如果旧容量和阈值都为0（表示使用默认值）
        //新容量为16，阈值为16*0.75
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    // 更新阈值
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    
    //创建新的Node数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    //指向新数组
    table = newTab;
    if (oldTab != null) {
        // 遍历旧数组
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            //如果头节点不为null
            if ((e = oldTab[j]) != null) {
                //设置为null，方便gc
                oldTab[j] = null;
                //如果只有头节点
                if (e.next == null)
                    //因为数组长度变化，所以重新计算位置，并为它在新数组分配
                    newTab[e.hash & (newCap - 1)] = e;
                //如果是树，进行分割操作
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { 
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 如果节点的hash值与旧容量进行与操作结果为0，放入低位链表
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 否则放入高位链表
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 将低位链表放入新数组对应位置
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 将高位链表放入新数组对应位置
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

resize的时机：元素个数超过阈值

HashMap默认容量为0，默认数组为null，阈值为0。第一次put时，设置容量为16，阈值为16*0.75=12。扩容大小为原来的2倍。



## get获取元素

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}


final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; 
    Node<K,V> first, e; 
    int n; K k;
    
    //如果数组不为空，对应hash位置的头节点不为空
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        //如果头节点满足hash、key相等条件，返回头元素
        if (first.hash == hash && 
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
         	//如果是树，在树中寻找
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            //如果是链表,循环直到链表结尾或者找到对应元素
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}    
```

若数组指定位置上为链表(元素数量小于8)，则查找元素的效率为O(n)。若是红黑树，则查找元素的效率为O(logn)。



## 总结

HashMap默认容量为0，默认数组为null，阈值为0。第一次put时，设置容量为16，阈值为16*0.75=12。扩容大小为原来的2倍。

put方法，会先根据hash&容量，计算出位置。若位置上无节点，则该元素为头节点，若已有链表，则从链表末尾插入(尾插法)，若链表元素大于等于8个，转换成红黑树。

存放元素时当数组的元素个数超过阈值时，就会扩容。会为每一个元素的hash与新容量进行计算，计算出在新数组中的位置。

获取元素时若数组指定位置上为链表(元素数量小于8)，则查找元素的效率为O(n)。若是红黑树，则查找元素的效率为O(logn)。
