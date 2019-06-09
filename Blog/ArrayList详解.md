ArrayList集合有3个构造函数：
```java
    //参数传入数组长度
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

    //初始化为默认空数组
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    //传入集合类型初始化
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```
添加元素的方法有4个：
```java
add(E e)
add(int index, E element)
addAll(Collection<? extends E> c)
addAll(int index, Collection<? extends E> c)
```


add(E e)方法相关的代码
```java
    protected transient int modCount = 0;

    //默认数组容量为10
    private static final int DEFAULT_CAPACITY = 10;

    //默认空数组
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    //ArrayList集合的数组(数组不能被序列化)
    transient Object[] elementData;

    //ArrayList的长度
    private int size;

    //添加数组元素
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);
        elementData[size++] = e;
        return true;
    }

    //如果数组未被初始化，则设置数组默认长度为10
    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        ensureExplicitCapacity(minCapacity);
    }

    //保证数组长度，如果需要添加元素位置超出数组长度则执行grow扩容
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

    //数组扩容，将原数组复制到新的数组达到扩大数组容量的目的
    private void grow(int minCapacity) {
        int oldCapacity = elementData.length;
        //以原数组长度的1.5倍进行扩容
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

add(int index, E element)相关方法：
```java
public void add(int index, E element) {
        rangeCheckForAdd(index);
        ensureCapacityInternal(size + 1);//上面有讲解，故省略
        System.arraycopy(elementData, index, elementData, index + 1,
                size - index);
        elementData[index] = element;
        size++;
    }

    //如果添加元素的索引小于0或者超出原数组的长度则抛出异常
    private void rangeCheckForAdd(int index) {
        if (index < 0 || index > this.size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
```

删除元素的方法有3个：
```java
remove(int index)
remove(Object o)
removeAll(Collection<?> c)
```

