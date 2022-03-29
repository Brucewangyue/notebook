# HashMap

 - Hash表(数组)长度：16
 - 最大Hash表长度：2^31
 - 加载因子：0.75f
 - 链表转树的阈值：8
 - 树转链表的阈值：6
 - 最小的树化的容量：64

## 线程不安全

hashmap多线程操作同时调用put()方法后可能导致get()死循环,从而使CPU使用率达到100%,从而使服务器宕机

多个线程put的时候造成了某个key值Entry key List的死循环，然后再调用put方法操作的时候就会进入链表的死循环内

**解决办法：HashTable、ConcurrentHashMap、Collections.synchronizedMap(hashMap)**  
HashTable和Vector 自带锁，现在都不用，JDK1.0就存在



## JDK1.7

采用头插法，将新元素插入到桶位的头部，将原头部的元素作为新元素的下一个链表元素

### 扩容机制

- 空参数的构造函数：以默认容量、默认负载因子、默认阈值初始化数组。内部数组是**空数组**
- 有参构造函数：根据参数确定容量、负载因子、阈值等
- 第一次put时会初始化数组，其容量变为**不小于指定容量的2的幂数**。然后根据负载因子确定阈值
- 如果不是第一次扩容，则 新容量=旧容量x2，新阈值=旧阈值x负载因子

## JDK1.8

1. 空参数的构造函数：实例化的HashMap默认内部数组是null，即没有实例化。第一次调用put方法时，则会开始第一次初始化扩容，长度为16
2. 有参构造函数：用于指定容量。会根据指定的正整数找到**不小于指定容量的2的幂数**，将这个数设置赋值给**阈值**（threshold）。第一次调用put方法时，会将阈值赋值给容量，然后让 阈值=容量x负载因子 。（因此并不是我们手动指定了容量就一定不会触发扩容，超过阈值后一样会扩容！！）
3. 如果不是第一次扩容，则容量变为原来的2倍，阈值也变为原来的2倍。*（容量和阈值都变为原来的2倍时，负载因子还是不变）*

此外还有几个细节需要注意：

- 首次put时，先会触发扩容（算是初始化），然后存入数据，然后判断是否需要扩容
- 不是首次put，则不再初始化，直接存入数据，然后判断是否需要扩容

### 内部是一个Node数组作为Hash表

```java
transient Node<K,V>[] table;
```

### hash算法

```java
// key的hash值 异或 key的hash值右移16位
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
// 作用：
// 从hash寻址算法中可以看出来，当数组长度比较短的时候（假如是16），无论hash值的高位（二进制的5位往左），如何变都没有影响到计算
// 通过以上的hash()可以让高位参与运算，让key能够在hash表中更散列
```

### 2的次方算法

指定一个数，计算出这个数最接近的下个2的次方的数，保证hash表的长度是2的次方

```java
// >>> 无符号右移
static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}

// in 5 out 8
// in 16 out 16
// in 29 out 32
```

**为什么数组长度一定要2的次方？**

这样做hash寻址的时候，会让数组下标不会超出范围，这样似的             直接用求余数的方式找下标更搞笑

### 构造函数

```java
  public HashMap(int initialCapacity, float loadFactor) {
      	// 参数校验
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
      
        // 让阈值始终是一个2的次方数
        this.threshold = tableSizeFor(initialCapacity);
    }
```

### put方法

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

// putVal
// onlyIfAbsent 如果是真则替换旧值
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    // 新的HashMap第一次put
    if ((tab = table) == null || (n = tab.length) == 0)
        // 创建Hash表，得到初始长度
        n = (tab = resize()).length;
    
    // hash寻址方式 (n - 1) & hash
    // 二进制与：同为1，结果为1
    // 假设hash值：0000 0000 0000 1010 0010 0100 0100 1010
    // n:16 :     0000 0000 0000 0000 0000 0100 0001 0000
    // n-1:       0000 0000 0000 0000 0000 0100 0000 1111
	// &:         0000 0000 0000 0000 0000 0100 0000 1010
    // 这里的关键点是，数组的长度必须是2的次方，这样减1再做与运算，就能限制数组的下标在有效范围内
    if ((p = tab[i = (n - 1) & hash]) == null) 
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

### resize方法

```java
// 假设 new HashMap(12,0.75f)  
// oldCap = 0
// threshold = 16

final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    
    // oldCap == 0
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    
    // oldCap == 0 and oldThr <= 0 
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    
    // 第一次put的时候 newThr == 0
    if (newThr == 0) {
        // ft = 16 * 0.75f = 12
        float ft = (float)newCap * loadFactor;
        // 12
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    
    threshold = newThr;
    
    // 下面是将旧的hash表中的所有元素 重新hash到新的hash表中
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
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

### JDK8的元素迁移

JDK8则因为巧妙的设计，性能有了大大的提升：由于数组的容量是以2的幂次方扩容的，那么一个Entity在扩容时，新的位置要么在**原位置**，要么在**原长度+原位置**的位置。原因如下图：

![image-20220329144046667](assets/image-20220329144046667.png)

数组长度变为原来的2倍，表现在二进制上就是**多了一个高位参与数组下标确定**。此时，一个元素通过hash转换坐标的方法计算后，恰好出现一个现象：最高位是0则坐标不变，最高位是1则坐标变为“10000+原坐标”，即“原长度+原坐标”。如下图：

![image-20220329144336035](assets/image-20220329144336035.png)

**因此，在扩容时，不需要重新计算元素的hash了，只需要判断最高位是1还是0就好了**