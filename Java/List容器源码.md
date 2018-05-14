## 一、ArrayList

ArrayList 是实现 List 接口的动态数组，所谓动态就是它的大小是可变的。实现了所有可选列表操作，并允许包括 null 在内的所有元素。除了实现 List 接口外，此类还提供一些方法来操作内部用来存储列表的数组的大小。

每个 ArrayList 实例都有一个容量，该容量是指用来存储列表元素的数组的大小。默认初始容量为 10。随着 ArrayList  中元素的增加，它的容量也会不断的自动增长。在每次添加新的元素时，ArrayList  都会检查是否需要进行扩容操作，扩容操作带来数据向新数组的重新拷贝，所以如果我们知道具体业务数据量，在构造 ArrayList 时可以给  ArrayList 指定一个初始容量，这样就会减少扩容时数据的拷贝问题。当然在添加大量元素前，应用程序也可以使用 ensureCapacity  操作来增加 ArrayList 实例的容量，这可以减少递增式再分配的数量。

**注意，ArrayList 实现不是同步的。**如果多个线程同时访问一个 ArrayList 实例，而其中至少一个线程从结构上修改了列表，那么它必须保持外部同步。所以为了保证同步，最好的办法是在创建时完成，以防止意外对列表进行不同步的访问：

List list = Collections.synchronizedList(new ArrayList(…));



### 底层使用数组

private transient Object[] elementData; ( transient 是在对对象做序列化时不会参与其中)



### 构造函数

ArrayList()：默认构造函数，提供初始容量为 10 的空列表。

ArrayList(int initialCapacity)：构造一个具有指定初始容量的空列表。

ArrayList(Collection<? extends E> c)：构造一个包含指定 collection 的元素的列表。



### 新增

ArrayList 提供了 add(E e)、add(int index, E element)、addAll(Collection<?  extends E> c)、addAll(int index, Collection<? extends E>  c)、set(int index, E element) 这个五个方法来实现 ArrayList 增加。 

```java
// 将指定的元素添加到此列表的尾部。
public boolean add(E e) {
    ensureCapacity(size + 1);  // 扩容检测
    elementData[size++] = e;
    return true;
}
```

```java
// 将指定的元素插入此列表中的指定位置。
// 在这个方法中最根本的方法就是 System.arraycopy() 方法，该方法的根本目的就是将 index 位置空出来以供新数据插入，这里需要进行数组数据的右移，这是非常麻烦和耗时的，所以如果指定的数据集合需要进行大量插入（中间插入）操作，推荐使用 LinkedList。

public void add(int index, E element) {
    // 判断索引位置是否正确
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(
            "Index: "+index+", Size: "+size);
    // 扩容检测
    ensureCapacity(size+1);  
    /*
     * 对源数组进行复制处理（位移），从index + 1到size-index。
     * 主要目的就是空出index位置供数据插入，
     * 即向右移动当前位于该位置的元素以及所有后续元素。 
     */
    System.arraycopy(elementData, index,  elementData, index + 1,
                 size - index);
    // 在指定位置赋值
    elementData[index] = element;
    size++;
}
```

```java
// 用指定的元素替代此列表中指定位置上的元素。
public E set(int index, E element) {
	//检测插入的位置是否越界
	RangeCheck(index);
	E oldValue = (E) elementData[index];
	//替代
	elementData[index] = element;
	return oldValue;
}
```



### 删除

ArrayList 提供了 remove(int index)、remove(Object o) （移除首次出现的o，如果存在的话）、removeRange(int fromIndex, int toIndex)、removeAll() 四个方法进行元素的删除。 



### 查找

ArrayList 提供了 get(int index) 用读取 ArrayList 中的元素。由于 ArrayList 是动态数组，所以我们完全可以根据下标来获取 ArrayList 中的元素，而且速度还比较快，故 ArrayList 长于随机访问。

```java
    public E get(int index) {
            RangeCheck(index);

            return (E) elementData[index];
        }
```



### 扩容

**ensureCapacity（）**方法就是 ArrayList 的扩容方法。在前面就提过 ArrayList  每次新增元素时都会需要进行容量检测判断，若新增元素后元素的个数会超过 ArrayList  的容量，就会进行扩容操作来满足新增元素的需求。所以当我们清楚知道业务数据量或者需要插入大量元素前，我可以使用 ensureCapacity  来手动增加 ArrayList 实例的容量，以减少递增式再分配的数量。 

```java
    public void ensureCapacity(int minCapacity) {
            //修改计时器
            modCount++;
            //ArrayList容量大小
            int oldCapacity = elementData.length;
            /*
             * 若当前需要的长度大于当前数组的长度时，进行扩容操 作
             */
            if (minCapacity > oldCapacity) {
                Object oldData[] = elementData;
                //计算新的容量大小，为当前容量的1.5倍
                int newCapacity = (oldCapacity * 3) / 2 + 1;
                if (newCapacity < minCapacity)
                newCapacity = minCapacity;
                //数组拷贝，生成新的数组
                elementData = Arrays.copyOf(elementData, newCapacity);
            }
        }
```

处理这个 ensureCapacity() 这个扩容数组外，ArrayList 还给我们提供了将底层数组的容量调整为当前列表保存的实际元素的大小的功能。它可以通过 trimToSize() 方法来实现。该方法可以最小化 ArrayList 实例的存储量。

```
    public void trimToSize() {
            modCount++;
            int oldCapacity = elementData.length;
            if (size < oldCapacity) {
                elementData = Arrays.copyOf(elementData, size);
            }
        }
```



## 二、LinkedList

LinkedList 与 ArrayList 一样实现 List 接口，只是 ArrayList 是 List  接口的大小可变数组的实现，LinkedList 是 List 接口链表的实现。基于链表实现的方式使得 LinkedList 在插入和删除时更优于 ArrayList，而随机访问则比 ArrayList 逊色些。 

除了实现 List 接口外，LinkedList 类还为在列表的开头及结尾 get、remove 和 insert 元素提供了统一的命名方法。这些操作允许将LinkedList用作堆栈、队列或双端队列。实现了 Deque 接口，为 add、poll 提供先进先出队列操作，以及其他堆栈和双端队列操作。所有操作都是按照双重链接列表的需要执行的。在列表中编索引的操作将从开头或结尾遍历列表（从靠近指定索引的一端）。

同时，与 ArrayList 一样此实现不是同步的。

### 属性

```java
    private transient Entry<E> header = new Entry<E>(null, null, null);// 双端链表结点
    private transient int size = 0;
```



### 

