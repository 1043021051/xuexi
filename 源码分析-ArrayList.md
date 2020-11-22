源码分析-ArrayList
====


一、说在前面
----
1.数组可能是我们最早接触到的数据结构之一，它是在内存中划分出一块连续的地址空间用来进行元素的存储，由于它直接操作内存，所以数组的性能要比集合类更好一些，这是使用数组的一大优势。但是我们知道数组存在致命的缺陷，就是在初始化时必须指定数组大小，并且在后续操作中不能再更改数组的大小。
2.在实际情况中我们遇到更多的是一开始并不知道要存放多少元素，而是希望容器能够自动的扩展它自身的容量以便能够存放更多的元素。``ArrayList``就能够很好的满足这样的需求，它能够自动扩展大小以适应存储元素的不断增加。它的底层是基于数组实现的，因此它具有数组的一些特点，例如查找修改快而插入删除慢。

二、分析
----
### List 集合
```
public interface List<E> extends Collection<E> {
    boolean add(E e);
    void add(int index, E element);
    boolean remove(Object o);
    E remove(int index);
    E set(int index, E element);
    E get(int index);
	...
}
```
### ArrayList 初始化及构造函数
```
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    private static final long serialVersionUID = 8683452581122892189L;

    /**
     * 默认初始化的容量
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * 初始化一个空实例的共享空数组实例。
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     *用于默认大小的空实例的共享空数组实例。我们将此与 empty _ elemtdata 区分开来, 以了解	 	  *添加分离第一元素时充气的程度
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
	 * 元素对象数据
     */
    transient Object[] elementData; // non-private to simplify nested class access

    /**
     * 当前 集合 的大小
     *
     * @serial
     */
    private int size;

    /**
     * 创建 ArrayList 的时候直接给元素对象开辟一个指定的容量大小
     * 否则 直接初始化一个空的元素对象
     *
     * @param 
     * @throws 
     *        
     */
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

    /**
     * 如果直接 new 一个空的构造函数的 ArrayList 内部直接创建一个默认空的对象数组
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    /**
     *
     * @param c 外部直接传入一个集合
     * @throws NullPointerException if the specified collection is null
     */
    public ArrayList(Collection<? extends E> c) {
    	//将外部传入进来的集合数据转化为数组赋值给 元素对象数组
        elementData = c.toArray();
        //更新集合大小
        if ((size = elementData.length) != 0) {
            // 判断引用的数据类型，并将引用转换成 Object 数组引用
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // 如果传入进来的是一个空的集合，那么直接创建一个空数组
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }

```

ArrayList 的内部存储结构就是一个 Object 类型的数组，因此它可以存放任意类型的元素。在构造ArrayList 的时候，如果传入初始大小那么它将新建一个指定容量的 Object 数组，如果不设置初始大小那么它将不会分配内存空间而是使用空的对象数组，在实际要放入元素时再进行内存分配。

三、集合底层数组扩容具体操作
----
### 1.添加 
```

/**
 * @param 将 e 元素添加到当前 elementData 中
 * @return 返回是否添加成功
 */
public boolean add(E e) {
    // 每次添加数据的时候都要检查当前数组容量是否需要扩容
    ensureCapacityInternal(size + 1);  
    // 直接赋值
    elementData[size++] = e;
    return true;
}
```
### 2.删除
```
public E remove(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    modCount++;
    E oldValue = (E) elementData[index];
	
    int numMoved = size - index - 1;
    if (numMoved > 0)
        //将index后面的元素向前挪动一位
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    // 置空引用
    elementData[--size] = null; 
	//返回被删除的元素
    return oldValue;
}

/**
 *
 * @param 删除外部传入的对象
 * @return 是否删除成功
 */
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        // 遍历元素 对象元素相等的话就直接删除
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

   /**
     * 删除全部
     * be empty after this call returns.
     */
    public void clear() {
        modCount++;

        // 直接把容量空间全部置为 NULL 交于 GC
        for (int i = 0; i < size; i++)
            elementData[i] = null;

        size = 0;
    }

```
### 3.改
```
/**
 * @return 返回旧元素
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public E set(int index, E element) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
	//取出要改变的旧元素
    E oldValue = (E) elementData[index];
    //将新元素更新
    elementData[index] = element;
    return oldValue;
}


```
### 4.查
```
/**
 * @param  根据索引直接从数据元素中获取
 * @return 返回取出的数据
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public E get(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    return (E) elementData[index];
}

```
### 5.数组扩容

```
/**
 *
 *
 * @param index 根据外部传入的 index 来进行存储数据
 * @param element element to be inserted
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public void add(int index, E element) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
	//检查是否需要扩容
    ensureCapacityInternal(size + 1);  
    //将需要插入的 index 位置往后移动一位
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    //在 数组的 index 位置直接存入 元素
    elementData[index] = element;
    size++;
}
private void ensureCapacityInternal(int minCapacity) {
	//如果当前元素数据为空的时候
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
    	// 和默认的扩容容量比较，取出最大值。
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
	// 数组初始化完毕进行下一步
    ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // 如果最小的扩容容量 比 当前的 元素数据还要大的时候进行扩容，下面是扩容的核心代码
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

	/**
	*
	*扩容核心代码
	*/
   private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        //扩容当前的容量的 1.5 倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        //如果新的容量大小小于最小的容量大小那么把当前最小的容量赋值给新的容量大小
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
            //主要检查新的容量是否已经超过最大容量了
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        //返回新扩容的元素数据
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }

```

四、总结:
----
每次添加元素前会调用 ensureCapacityInternal 这个方法进行集合容量检查。在这个方法内部会检查当前集合的内部数组是否还是个空数组，如果是就新建默认大小为 10 的 Object 数组。如果不是则证明当前集合已经被初始化过，那么就调用 ensureExplicitCapacity 方法检查当前数组的容量是否满足这个最小所需容量，不满足的话就调用 grow 方法进行扩容。在 grow 方法内部可以看到，每次扩容都是增加原来数组长度的一半，扩容实际上是新建一个容量更大的数组，将原先数组的元素全部复制到新的数组上，然后再抛弃原先的数组转而使用新的数组。

至此，我们对 ArrayList 中比较常用的方法做了分析，其中总结要点：

1.ArrayList 底层实现是基于数组的，因此对指定下标的查找和修改比较快，但是删除和插入操作比较慢。
2.构造 ArrayList 时尽量指定容量，减少扩容时带来的数组复制操作，如果不知道大小可以赋值为默认容量 10。
3.每次添加元素之前会检查是否需要扩容，每次扩容都是增加原有容量的一半。
4.每次对下标的操作都会进行安全性检查，如果出现数组越界就立即抛出异常。
5.ArrayList 的所有方法都没有进行同步，因此它不是线程安全的。




