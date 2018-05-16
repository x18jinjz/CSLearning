[TOC]



## 一、HashMap

### 数据结构

数组 + 链表 + 红黑树（默认一个桶上结点数超过8时转红黑树，低于6时转链表）

![这里写图片描述](https://img-blog.csdn.net/20170804142150998?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcWF6d3lj/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 类属性

```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable {

    private static final long serialVersionUID = 362498820763181265L;    // 序列号
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;   // 默认的初始容量是16
    static final int MAXIMUM_CAPACITY = 1 << 30;     // 最大容量
    static final float DEFAULT_LOAD_FACTOR = 0.75f;     // 默认的填充因子
    static final int TREEIFY_THRESHOLD = 8;     // 当桶(bucket)上的结点数大于这个值时会转成红黑树
    static final int UNTREEIFY_THRESHOLD = 6;    // 当桶(bucket)上的结点数小于这个值时树转链表
    static final int MIN_TREEIFY_CAPACITY = 64;    // 桶中结构转化为红黑树对应的数组的最小大小，如果当前容量小于它，就不会将链表转化为红黑树，而是用resize()代替
    
    transient Node<k,v>[] table;     // 存储元素的数组，总是2的幂
    transient Set<map.entry<k,v>> entrySet;    // 存放具体元素的集
    transient int size;    // 存放元素的个数，不等于数组的长度。
    transient int modCount;     // 每次扩容和更改map结构的计数器
    int threshold;    // 临界值 当实际节点个数超过临界值(容量*填充因子)时，会进行扩容
    final float loadFactor;    // 填充因子
}
```



### 类的构造函数

在HashMap的构造函数中并没有对数组`Node<K,V>[] table`初始化，而是简单的设置参数，在首次put时调用resize()分配内存。

```java
// 指定初始容量和填充因子
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                            initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                            loadFactor);
    
    this.loadFactor = loadFactor;
    // 通过tableSizeFor(cap)计算出不小于initialCapacity的最近的2的幂作为初始容量，将其先保存在threshold里，当put时判断数组为空会调用resize分配内存，并重新计算正确的threshold
    this.threshold = tableSizeFor(initialCapacity);   
} 

// 指定初始容量
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

//默认构造函数
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; 
}

//HashMap(Map<? extends K>)型构造函数
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    // 将m中的所有元素添加至HashMap中
    putMapEntries(m, false);
}
```

其中tableSizeFor(initialCapacity)返回最近的不小于输入参数的2的整数次幂。比如10，则返回16。

```java
static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
/* 原理如下：
5个移位操作，会使cap的二进制从最高位的1到末尾全部置为1。
假设cap的二进制为01xx…xx。
对cap右移1位：01xx…xx，位或：011xx…xx，使得与最高位的1紧邻的右边一位为1，
对cap右移2位：00011x..xx，位或：01111x..xx，使得从最高位的1开始的四位也为1，
以此类推，int为32位，所以在右移16位后异或最多得到32个连续的1，保证从最高位的1到末尾全部为1。
最后让结果+1，就得到了最近的大于cap的2的整数次幂。
让cap-1再赋值给n的目的是令找到的目标值大于或等于原值。如果cap本身是2的幂，如8（1000(2)），不对它减1而直接操作，将得到16。
*/
```



### Hash实现

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
/*没有直接使用key的hashcode()，而是使key的hashcode()高16位不变，低16位与高16位异或作为最终hash值。
```

![这里写图片描述](https://img-blog.csdn.net/20170804153440578?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcWF6d3lj/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast) 



### 求元素在数组中的位置

return (length - 1) & hash  因为length总是为2的n次方，效果与对length取余相同。



### 扩容 resize（）

resize用来重新分配内存 
 \+ 当数组未初始化，按照之前在threashold中保存的初始容量分配内存，没有就使用缺省值 
 \+ 当超过限制时，就扩充两倍，因为我们使用的是2次幂的扩展，所以，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置

如oldCap为16时

![这里写图片描述](https://img-blog.csdn.net/20170804153539203?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcWF6d3lj/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

因此，我们在扩充HashMap的时候，不需要重新计算hash，只需要看看原来的hash值高位新增的那个bit是1还是0，是0的话索引不变，是1的话索引变成“原索引+oldCap”, 直接拆分原链表为高低链表相比先保存数据再寻址追加效率更好。

```java
final Node<K,V>[] resize() {
    // 保存当前信息
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    
    int newCap, newThr = 0;
    // 之前table大小大于0，即已初始化
    if (oldCap > 0) {
        // 超过最大值就不再扩充了，只设置阈值
        if (oldCap >= MAXIMUM_CAPACITY) {
            // 阈值为最大整形
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 容量翻倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
            oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 阈值翻倍
            newThr = oldThr << 1; // double threshold
    }
    
    else if (oldThr > 0)            // 数组没初始化且阈值已确定
        newCap = oldThr;
    
    else {        // 使用缺省值（使用默认构造函数初始化）
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 计算新阈值
    if (newThr == 0) {  // 上面的else if (oldThr > 0)
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    // 初始化table
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    // 之前的table已经初始化过
    if (oldTab != null) {
        // 复制元素，重新进行hash
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)         //桶中只有一个结点
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)         //红黑树
                    //根据(e.hash & oldCap)分为两个，如果哪个数目不大于UNTREEIFY_THRESHOLD，就转为链表
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null; // 对应新增高位为0的链表，放在原位置
                    Node<K,V> hiHead = null, hiTail = null; // 对应新增高位为0的链表，放在原下标+oldCap
                    Node<K,V> next;
                    // 将同一桶中的元素根据(e.hash & oldCap)是否为0进行分割成两个不同的链表，完成rehash
                    do {
                        next = e.next;//保存下一个节点
                        if ((e.hash & oldCap) == 0) {   //上述所讲方法，判断新增的高位为0或1决定新位置
                            if (loTail == null)//第一个结点让loTail和loHead都指向它
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {                                      //hash到高部分即原索引+oldCap
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



### 插入 put（）

1. 对key的hashCode()做hash，然后再计算桶的index;
2. 如果没碰撞直接放到桶bucket里；
3. 如果碰撞了，以链表的形式存在buckets后；
4. 如果碰撞导致链表过长(大于等于TREEIFY_THRESHOLD)，就把链表转换成红黑树（若数组容量小于MIN_TREEIFY_CAPACITY，不进行转换而是进行resize操作，这一步判断在treeifyBin(）红黑树的操作中）；
5. 如果节点已经存在就替换old value(保证key的唯一性)；
6. 如果表中实际元素个数超过阈值(超过load factor*current capacity)，就要resize。

```java
public V put(K key, V value) {
    // 对key的hashCode()做hash
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // table未初始化或者长度为0，进行扩容，n为桶的个数
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    if ((p = tab[i = (n - 1) & hash]) == null) //(n - 1) & hash确定元素所在桶，为空则生成结点放入桶中
        tab[i] = newNode(hash, key, value, null);
    else {    // 桶中已经存在元素
        Node<K,V> e; K k;
        if (p.hash == hash &&    // 比较桶中第一个元素的hash值相等，key相等
            ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
                
        // hash值不相等或key不相等
        else if (p instanceof TreeNode)  //红黑树
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value); // 放入树中
        
        else { // 为链表结点
            for (int binCount = 0; ; ++binCount) {    
                if ((e = p.next) == null) {    // 到达链表的尾部,在尾部插入新结点
                    p.next = newNode(hash, key, value, null);
                    // 结点数量达到阈值，调用treeifyBin()做进一步判断是否转为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&  // 链表中结点的key值与插入的元素的key值相等，跳出循环
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // 表示在桶中找到key值、hash值与插入元素相等的结点
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;	//用新值替换旧值
            // 访问后回调
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold) // 实际大小大于阈值则扩容
        resize();
    // 插入后回调
    afterNodeInsertion(evict);
    return null;
}

//将指定映射的所有映射关系复制到此映射中
public void putAll(Map<? extends K, ? extends V> m) {
    putMapEntries(m, true);
}

//将m的所有元素存入本HashMap实例中，evict为false时表示构造初始HashMap
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
        // table未初始化
        if (table == null) { // pre-size
            //计算初始容量
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                    (int)ft : MAXIMUM_CAPACITY);
            if (t > threshold)
                threshold = tableSizeFor(t);//同样先保存容量到threshold
        }
        // 已初始化，并且m元素个数大于阈值，进行扩容处理
        else if (s > threshold)
            resize();
        // 将m中的所有元素添加至HashMap中
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}

//将链表转换为红黑树
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    //若数组容量小于MIN_TREEIFY_CAPACITY，不进行转换而是进行resize操作
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);//将Node转换为TreeNode
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);//重新排序形成红黑树
    }
}
```



### 获取 get（）

get函数大致思路如下： 
 \1. bucket里的第一个节点，直接命中； 
 \2. 如果有冲突，则通过key.equals(k)去查找对应的entry，若为树，复杂度O(logn)， 若为链表，O(n)

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // table已经初始化，长度大于0，且根据hash寻找table中的项也不为空
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 比较桶中第一个节点
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 桶中不止一个结点
        if ((e = first.next) != null) {
            // 为红黑树结点
            if (first instanceof TreeNode)
                // 在红黑树中查找
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 否则，在链表中查找
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



### containsKey（）和 containsValue（）

```java
public boolean containsKey(Object key) {
    return getNode(hash(key), key) != null;
}

public boolean containsValue(Object value) {
    Node<K,V>[] tab; V v;
    if ((tab = table) != null && size > 0) {
        //外层循环搜索数组
        for (int i = 0; i < tab.length; ++i) {
            //内层循环搜索链表
            for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                if ((v = e.value) == value ||
                    (value != null && value.equals(v)))
                    return true;
            }
        }
    }
    return false;
}
```



### 删除 remove（）

```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

/**
 * 用于实现 remove()方法和其他相关的方法
 *
 * @param hash 键的hash值
 * @param key 键
 * @param value the value to match if matchValue, else ignored
 * @param matchValue if true only remove if value is equal
 * @param movable if false do not move other nodes while removing
 * @return the node, or null if none
 */
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    //table数组非空，键的hash值所指向的数组中的元素非空
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;  // node指向最终的结果结点，e为链表中的遍历指针
        if (p.hash == hash &&   // 检查第一个节点
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        // 如果第一个节点不匹配
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)  //树
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        // 判断是否存在，如果matchValue为true，需要比较值是否相等
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)   // 树
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)  // 匹配第一个节点
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}

public void clear() {
    Node<K,V>[] tab;
    modCount++;
    if ((tab = table) != null && size > 0) {
        size = 0;
        for (int i = 0; i < tab.length; ++i)
            tab[i] = null;
    }
}
```


## 二、LinkedHashMap

`LinkedHashMap` 是一个**关联数组、哈希表**，它是**线程不安全**的，允许**key为null**,**value为null**。

它继承自`HashMap`，实现了`Map<K,V>`接口。其内部还维护了一个**双向链表**，在每次**插入数据，或者访问、修改数据**时，**会增加节点、或调整链表的节点顺序**。以决定迭代时输出的顺序。

默认情况，遍历时的顺序是**按照插入节点的顺序**。这也是其与`HashMap`最大的区别。也可以在构造时传入`accessOrder`参数，使得其遍历顺序**按照访问的顺序**输出。

### 属性

![这里写图片描述](https://img-blog.csdn.net/20161105174920486) 

```java
//LinkedHashMap的链表节点继承了HashMap的节点，而且每个节点都包含了前指针和后指针，所以这里可以看出它是一个双向链表
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}

//头指针
transient LinkedHashMap.Entry<K,V> head;

//尾指针
transient LinkedHashMap.Entry<K,V> tail;

//默认为false。当为true时，表示链表中键值对的顺序与每个键的插入顺序一致（false时重复插入不会覆盖），也就是说重复插入键或访问值，也会更新顺序
final boolean accessOrder;
```



### 顺序相关方法

HashMap中有如下三个方法，有不少地方还调用过它们。其实这三个方法表示的是在访问、插入、删除某个节点之后，进行一些处理，它们在LinkedHashMap都有各自的实现。LinkedHashMap正是通过重写这三个方法来保证链表的插入、删除的有序性：

```java
// Callbacks to allow LinkedHashMap post-actions
void afterNodeAccess(Node<K,V> p) { }
void afterNodeInsertion(boolean evict) { }
void afterNodeRemoval(Node<K,V> p) { }
```

**afterNodeAccess方法**

```java
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    //当accessOrder的值为true，且e不是尾节点
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```

这段代码的意思简洁明了，就是把当前节点e移至链表的尾部。因为使用的是双向链表，所以在尾部插入可以以O（1）的时间复杂度来完成。并且只有当`accessOrder`设置为true时，才会执行这个操作。在HashMap的`putVal`方法中，就调用了这个方法。

**afterNodeInsertion方法**

```java
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}
//LinkedHashMap 默认返回false 则不删除节点。 返回true 代表要删除最早的节点。通常构建一个LruCache会在达到Cache的上限是返回true
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```

`void afterNodeInsertion(boolean evict)`以及`boolean removeEldestEntry(Map.Entry<K,V> eldest)`是构建LRUCache需要的回调，在LinkedHashMap里可以忽略它们。 

**afterNodeRemoval方法**

```java
void afterNodeRemoval(Node<K,V> e) { // unlink
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    p.before = p.after = null;
    if (b == null)
        head = a;
    else
        b.after = a;
    if (a == null)
        tail = b;
    else
        a.before = b;
}
```

这个方法是当HashMap删除一个键值对时调用的，它会把在HashMap中删除的那个键值对一并从链表中删除，保证了哈希表和链表的一致性。



### 获取 get（）

```java
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
```

调用的是HashMap的`getNode`方法来获取结果的。并且，如果你把`accessOrder`设置为true，那么在获取到值之后，还会调用`afterNodeAccess`方法。



### **put（）和remove（）**

没有重写put方法，所以调用的`put`方法其实是HashMap的`put`方法。因为HashMap的put方法中调用了`afterNodeAccess`方法和`afterNodeInsertion`方法，已经足够保证链表的有序性了，所以它也就没有重写put方法了。remove方法也是如此。



## 三、ConcurrentHashMap

## JDK 7

在JDK6中，ConcurrentHashMap将数据分成一段一段存储，给每一段数据配一把锁，当一个线程获得锁互斥访问一个段数据时，其他段的数据也可被其他线程访问；每个Segment拥有一把**可重入锁**。segment数组的大小决定了并发度 。

每一个segment都是一个`HashEntry<K,V>[] table`， table中的每一个元素本质上都是一个HashEntry的单向队列（单向链表实现）。每一个segment都是一个`HashEntry<K,V>[] table`， table中的每一个元素本质上都是一个HashEntry的单向队列。 

**锁分离**：当一个线程访问Node/键值对数据时，必须获得与它对应的segment锁，其他线程可以访问其他Segment中的数据（锁分离）； 

![img](https://upload-images.jianshu.io/upload_images/7853175-5137bd1f4f444bde.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)  



## JDK 8

JDK 1.8取消类`segments`字段，直接用table数组存储键值对，JDK1.6中每个bucket中键值对组织方式是单向链表，查找复杂度是`O(n)`，JDK1.8中当链表长度超过`TREEIFY_THRESHOLD`时，链表转换为红黑树，查询复杂度可以降低到O(log n)，改进性能； 

![HashMap结构](https://segmentfault.com/img/remote/1460000009001472?w=630&h=567) 

### 锁分离

JDK8中，一个线程每次对一个桶（链表 or 红黑树）进行加锁，其他线程仍然可以访问其他桶。

### 线程安全

ConcurrentHashMap底层数据结构与HashMap相同，仍然采用**table数组+链表+红黑树**结构。
一个线程进行put/remove操作时，对桶（链表 or 红黑树）加上synchronized独占锁；底层**采用CAS算法保证线程安全**。

### 部分属性和内部类

```java
private static final int DEFAULT_CAPACITY = 16;            //默认bin数组的大小，必须为2的n次方  
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;   //在java 8中已经不再使用这个量了，只是为了向前兼容才保留的  
private static final float LOAD_FACTOR = 0.75f;            //默认装填因子  
static final int TREEIFY_THRESHOLD = 8;     //当bin中大于8个node时，将此bin由list式转换为tree式  
static final int UNTREEIFY_THRESHOLD = 6;   //逆tree化  
static final int MIN_TREEIFY_CAPACITY = 64; //大于64个bucket了，才会考虑tree化  
static final int NCPU = Runtime.getRuntime().availableProcessors();   //取得CPU个数  
transient volatile Node<K,V>[] table;        //与HashMap相比，增加了volatile声明  
private transient volatile long baseCount;   //base counter，经由CAS来update  
```

```
static class Node<K,V> implements Map.Entry<K,V> {  
        final int hash;  
        final K key;                //注意下面两行，与HashMap不一样：  
        volatile V val;             //  V value;  
        volatile Node<K,V> next;    //  Node<K,V> next;  
                                    //将value与next均设为volatile，为了保证可见性  
        ... ...                     //Node中并无synchronized方法  
    } 
```



### 初始化

```java
public ConcurrentHashMap() {

}

public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;

}
// sizeCtl = (1.5 * initialCapacity + 1)，然后向上取最近的 2 的 n 次方】。如 initialCapacity 为 10，那么得到 sizeCtl 为 16，如果 initialCapacity 为 11，得到 sizeCtl 为 32。
```

### Table初始化

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0) 	// 其他线程正在初始化
            Thread.yield(); 	// lost initialization race; just spin
        					   // 自旋，yield表示人为提示CPU进行切换其他线程，也可能切回来
        
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { // CAS将sizeCtl设置为-1，代表抢到锁
            try {
                if ((tab = table) == null || tab.length == 0) {
                    // DEFAULT_CAPACITY 默认初始容量是 16
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    // 初始化数组，长度为 16 或初始化时提供的长度
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    // 将这个数组赋值给 table，table 是 volatile 的
                    table = tab = nt;
                    // 如果 n 为 16 的话，那么这里 sc = 12
                    // 其实就是 0.75 * n
                    sc = n - (n >>> 2);
                }
            } finally {
                // 设置 sizeCtl 为 sc，我们就当是 12 吧
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```



### 插入 put（）

```java
public V put(K key, V value) {  
    return putVal(key, value, false);  
}  

//调整hash值分布的方法，类似于HashMap中的hash(Object key)方法  
static final int spread(int h) {  
    return (h ^ (h >>> 16)) & HASH_BITS;  
}  

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException(); // ConcurrentHashMap不允许null key或null value  
    int hash = spread(key.hashCode());
    int binCount = 0;	// 记录bucket中的node的数量  
    
    for (Node<K,V>[] tab = table;;) { // 不断CAS探测，如果其他线程正在修改tab，CAS尝试失败，直到成功为止
        Node<K,V> f; int n, i, fh;
        
        if (tab == null || (n = tab.length) == 0) // 如果空表，对tab进行初始化
            tab = initTable();

        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { // bucket为空   
            /* 如果 CAS 失败，那就是有并发操作，进到下一个循环
            * 如果bucket为空，则
            * 不加锁，用CAS添加新键值对
            */
            if (casTabAt(tab, i, null,new Node<K,V>(hash, key, value, null)))  
                break;                   							
        }
        // 检测到tab[i]桶正在进行rehash,
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 对桶的首元素上锁独占
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    // 桶中键值对组织形式是链表
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                // 查找到对应键值对，更新值
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            // 桶中没有对应键值对，插入到链表尾部
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    // 桶中键值对组织形式是红黑树
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            // 检查桶中键值对总数
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    // 链表转换为红黑树
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 更新baseCount
    addCount(1L, binCount);
    return null;
}
```



### 获取 get（）

1.计算 hash 值
2.根据 hash 值找到数组对应位置: (n – 1) & h
3.根据该位置处结点性质进行相应查找
    a.如果该位置为 null，那么直接返回 null 就可以了
    b.如果该位置处的节点刚好就是我们需要的，返回该节点的值即可
    c.如果该位置节点的 hash 值小于 0，说明正在扩容，或者是红黑树，后面我们再介绍 find 方法
    d.如果以上 3 条都不满足，那就是链表，进行遍历比对即可

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {	// bucket非空
        if ((eh = e.hash) == h) {	
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))  // 头结点匹配
                return e.val;
        }
        else if (eh < 0)	// 头结点的hash小于 0，说明正在扩容，或者该位置是红黑树
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {		// 遍历链表
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```



### 扩容 tryPresize（）

```java
// 方法参数 size 传进来的时候就已经翻了倍了
private final void tryPresize(int size) {
    // c：size 的 1.5 倍，再加 1，再往上取最近的 2 的 n 次方。
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    while ((sc = sizeCtl) >= 0) {	// 没有其他线程正在初始化或扩容
        Node<K,V>[] tab = table; int n;
         if (tab == null || (n = tab.length) == 0) {	
            n = (sc > c) ? sc : c;
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { // 期间没有其他线程对表操作，则CAS将SIZECTL状态置为-1，表示正在进行初始化 
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2); // 0.75 * n 
                    }
                } finally {
                    sizeCtl = sc;  // 更新扩容阀值
                }
            }
        }
        else if (c <= sc || n >= MAXIMUM_CAPACITY)	// 若欲扩容值<=原阀值，或现有容量>=最打值，空操作
            break;
        else if (tab == table) {	// table不为空，且在此期间其他线程未修改table 
            int rs = resizeStamp(n);
            if (sc < 0) {
                Node<K,V>[] nt;
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))// 2. 用 CAS 将 sizeCtl 加 1，然后执行 transfer 方法  此时 nextTab 不为 null 
                    transfer(tab, nt);
            }
            // 1. 将 sizeCtl 设置为 (rs << RESIZE_STAMP_SHIFT) + 2)
            //     我是没看懂这个值真正的意义是什么？不过可以计算出来的是，结果是一个比较大的负数
            //  调用 transfer 方法，此时 nextTab 参数为 null
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}
```

**TODO**：`transfer`待续 .........