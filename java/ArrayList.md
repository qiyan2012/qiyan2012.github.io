

# ArrayList



日常工作中用的最多的集合类。



## 初始化

底层是Object数组，初始化时若不指定容量或容量为0，则默认是空数组。若指定大于0的容量，则初始化为指定容量的数组。

也就是说当我们使用`new ArrayList<>()`时，内部的容量是0。

```java
private static final int DEFAULT_CAPACITY = 10;

private static final Object[] EMPTY_ELEMENTDATA = {};

private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

transient Object[] elementData; 

private int size;

public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
```



## add添加元素

add()方法添加元素时，首先要保证数组大小足够，并使modCount++，然后在下一个位置赋值。

```java
public boolean add(E e) {
    //确保容量足够，并使modCount++
    ensureCapacityInternal(size + 1);  
    //赋值
    elementData[size++] = e;
    return true;
}

private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

//默认容量
private static final int DEFAULT_CAPACITY = 10;

//若数组是空数组，返回max(10,minCapacity)。
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}

//modCount++，若所需最小容量大于数组长度，扩容
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    if (minCapacity - elementData.length > 0)
        //扩容
        grow(minCapacity);
}
```



modCount表示集合的修改次数，在线程不安全的集合中，使用modCount来校验线程安全。



## 扩容

```java
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

private void grow(int minCapacity) {
    // 原容量
    int oldCapacity = elementData.length;
    // 新容量为原容量的1.5倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    //若新容量还不够，则把新容量设置成需要的最小容量
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;

    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    
    // 扩容并拷贝原数组
    elementData = Arrays.copyOf(elementData, newCapacity);
}

```

- 扩容的触发条件是数组全部被占满
- 扩容是以旧容量的1.5倍扩容，并不是2倍扩容
- 添加元素时，没有对元素校验，允许为null，也允许元素重复。





## remove删除元素

```java
public E remove(int index) {
    //判断下表是否越界
    rangeCheck(index);
	
    modCount++;
    //获得要删除下标的元素，做返回用
    E oldValue = elementData(index);
	
    //需要移动的元素数量
    int numMoved = size - index - 1;
    //从下标处开始拷贝numMoved个元素，也就是后面的元素整体向左移动一个位置
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    //清理最后一个位置的元素，帮助gc
    elementData[--size] = null; 
	//返回旧值
    return oldValue;
}

private void rangeCheck(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

E elementData(int index) {
    return (E) elementData[index];
}

public boolean remove(Object o) {
    //判断是否为null
    if (o == null) {
        //找到第一个null，删除返回
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        //找到第一个值为o的，删除返回
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```



## removeAll批量删除

```java
public boolean removeAll(Collection<?> c) {
    //校验非空
    Objects.requireNonNull(c);
    return batchRemove(c, false);
}

private boolean batchRemove(Collection<?> c, boolean complement) {
    final Object[] elementData = this.elementData;
    //r:当前位置 w:需要保留元素的位置
    int r = 0, w = 0;
    boolean modified = false;
    try {
        for (; r < size; r++)
            //如果不存在，则需要该元素
            if (c.contains(elementData[r]) == complement)
                //把需要保留的元素左移
                elementData[w++] = elementData[r];
    } finally {
        // 当出现异常情况的时候（并发修改），可能不相等
        if (r != size) {
            // 可能是其它线程添加了元素，把新增的元素也添加到新的数组中
            System.arraycopy(elementData, r,
                             elementData, w,
                             size - r);
            w += size - r;
        }
        // 把不需要保留的元素设置为null
        if (w != size) {
            for (int i = w; i < size; i++)
                elementData[i] = null;
            modCount += size - w;
            size = w;
            modified = true;
        }
    }
    return modified;
}
```

## set

修改指定下标的值

```java
public E set(int index, E element) {
    //下标检测
    rangeCheck(index);
	
    //获取旧值，返回用
    E oldValue = elementData(index);
    //更新值
    elementData[index] = element;
    return oldValue;
}

private void rangeCheck(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```



## 并发修改

```java
public static void main(String[] args) {
    List<String> list = new ArrayList<>();
    list.add("1");
    list.add("2");
    list.add("3");
    Iterator<String> iterator = list.iterator();
    while (iterator.hasNext()) {
        String nextElement = iterator.next();
        if (Integer.parseInt(nextElement) < 2) {
            list.add("2");
        }
    }
}
```

运行此程序，控制台输出，程序出现异常：

```java
Exception in thread "main" java.util.ConcurrentModificationException
```

可见，控制台显示的ConcurrentModificationException，即并发修改异常。



ArrayList会有一个`modCount`，表示list的变更次数。在获取迭代器时，会有一个`expectedModCount`，表示期望的变更次数。在迭代时，会校验`expectedModCount`和`modCount`是否相同，若不同会抛出`ConcurrentModificationException`异常。

这是一种快速失败的思想。

> 快速失败(fail-fast)
>
> 这是一种理念，说白了就是在做系统设计的时候先考虑异常情况，一旦发生异常，直接停止并上报。
>
> 举一个最简单的fail-fast的例子：
>
> ```java
> public int divide(int divisor,int dividend){
>     if(divisor == 0){
>         throw new RuntimeException("divisor can't be null");
>     }
>     return dividend/divisor;
> }
> ```
>
> 在divide方法中，我们对除数做了个简单的检查，如果其值为0，那么就直接抛出一个异常，并明确提示异常原因。这其实就是fail-fast理念的实际应用。
>
> 这样做的好处就是可以预先识别出一些错误情况，一方面可以避免执行复杂的其他代码，另外一方面，这种异常情况被识别之后也可以针对性的做一些单独处理。



ArrayList中add、remove等方法都会修改`modCount`,但set不会。

可以使用迭代器的remove方法删除元素，会同时修改exceptedModCount

```java
//内部类Itr的remove()方法
 public void remove() {
	if (lastRet < 0)
	     throw new IllegalStateException();
	 checkForComodification();
	
	 try {
	 	//其底层调用的还是ArrayList的remove()方法
	     ArrayList.this.remove(lastRet);
	     cursor = lastRet;
	     lastRet = -1;
	     //重点在这里，每删除一个元素会把实际修改次数赋值给预期
	     //修改次数，即两者总是相等的
	     expectedModCount = modCount;
	 } catch (IndexOutOfBoundsException ex) {
	     throw new ConcurrentModificationException();
	 }
}
```

