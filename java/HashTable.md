

# HashTable



从没用过，但面试可能会问。



## 构造方法









## put

```java
public synchronized V put(K key, V value) {
    if (value == null) {
        throw new NullPointerException();
    }

    Entry<?,?> tab[] = table;
    //计算hash和位置
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    //判断当前位置是否由key、hash相同的节点，是更新还是新增
    Entry<K,V> entry = (Entry<K,V>)tab[index];
    for(; entry != null ; entry = entry.next) {
        if ((entry.hash == hash) && entry.key.equals(key)) {
            V old = entry.value;
            entry.value = value;
            return old;
        }
    }

    addEntry(hash, key, value, index);
    return null;
}

private void addEntry(int hash, K key, V value, int index) {
    modCount++;

    Entry<?,?> tab[] = table;
    //如果元素数量大于阈值
    if (count >= threshold) {
        //扩容、对所有节点重新hash和容量大小计算出新位置并设置
        rehash();
        tab = table;
        hash = key.hashCode();
        index = (hash & 0x7FFFFFFF) % tab.length;
    }

    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>) tab[index];
    //头插法，最后一个入参是指向下一节点的指针。
    tab[index] = new Entry<>(hash, key, value, e);
    //元素个数增加
    count++;
}
```

可以看到和HashMap的原理差不多，但是加了synchronized来保证线程安全。同时不支持红黑树。

HashTable中value为null会抛错，同时调用了key.hashCode()，说明key也不允许为null，null.hashCode会报空指针。(HashMap支持key和value为null，key为null的话，hash为0)





## get





