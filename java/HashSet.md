

# HashSet



HashSet由HashMap实现。



## 构造方法

```java
private transient HashMap<E,Object> map;

//做map的value
private static final Object PRESENT = new Object();

public HashSet() {
    map = new HashMap<>();
}

public HashSet(Collection<? extends E> c) {
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}


public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}


public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}


HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```

可以看到HashSet的初始化其实就是初始化一个HashMap





## add、remove

```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}


public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}
```

add和remove也是使用HashMap的add和remove



这样也就可以明白，HashSet之所以元素不重复是因为HashMap的key不重复，无序也是因为HashMap无序。
