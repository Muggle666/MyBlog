# 简介

ArrayList集合类是基于数组实现的，其数组容量的大小可以动态的变化，数组的元素可以为null。值得注意的是，ArrayList中所有的方法都是非线程安全的！在多线程环境下可以使用synchronized或者使用显式锁，当然还有一个更加便捷的方式，就是使用Collections.synchronizedList(List<T> list)方法，这样的话，使用所有的ArrayList方法都是线程安全的。

```java
List list = Collections.synchronizedList(new ArrayList());
```

## 源码剖析

#### 1.ArrayList集合类的结构图
![ArrayList结构图](https://raw.githubusercontent.com/MuggleLee/PicGo/master/ArrayList/ArrayList%E7%BB%93%E6%9E%84%E5%9B%BE.png)

ArrayList集合有3个构造函数：
```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

ArrayList集合类继承了AbstractList抽象类和实现了List接口，拥有了新增，删除，修改和遍历的功能；实现RandomAccess接口，这是一个标志接口，只要有类实现该接口就代表可以快速随机访问元素；实现Cloneable接口和Serializable接口可以克隆和序列化。


#### 2.ArrayList集合类的构造函数：
```java
    //参数传入数组长度，可自定义数组大小
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

#### 3.添加元素——add()方法的讲解和ArrayList动态扩容机制
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

#### 4.删除元素——remove()
删除的方法有以下3个：

```java
remove(int index)
remove(Object o)
removeAll(Collection<?> c)
```

```java
    public E remove(int index) {
        rangeCheck(index);//判断是否超出索引长度

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                    numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }

    public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }

    public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);//判断对象是否是null
        return batchRemove(c, false);
    }

    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                    numMoved);
        elementData[--size] = null; 
    }

    private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;
        int r = 0, w = 0;
        boolean modified = false;
        try {
            for (; r < size; r++)
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
        } finally {
            // Preserve behavioral compatibility with AbstractCollection,
            // even if c.contains() throws.
            if (r != size) {
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                w += size - r;
            }
            if (w != size) {
                // clear to let GC do its work
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

通过源码可以知道，删除元素都要进行数组的重组，而且当被删除的元素越靠近集合数组的前面，数组的重组开销就越大。

#### 5.获取元素——get()

由于ArrayList集合类实现了RandomAccess接口，所以可以快速的获取元素。

```java
    E elementData(int index) {
        return (E) elementData[index];
    }


    public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }
```

#### 6.为ArrayList集合类“瘦身”——trimToSize()：将数组的容量设置size，省去多余空间，优化程序。

![trimToSize()方法示意图](https://raw.githubusercontent.com/MuggleLee/PicGo/master/ArrayList/ArrayList_trimToSize()_sample.png)

```java
    public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA // 当数组长度为0，设置为EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size); // 重组数组容量，数组容量大小为size
        }
    }
```


#### 7.指定ArrayList集合类数组容量大小——ensureCapacity(int minCapacity)

```java
    public void ensureCapacity(int minCapacity) {
    	// 如果集合数组不是默认的空数组则设置为0，否则设置为默认数组长度
        int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) ? 0 : DEFAULT_CAPACITY;
        if (minCapacity > minExpand) {
            ensureExplicitCapacity(minCapacity); // 保证数组长度，如果minCapacity超出数组长度则执行grow()方法扩容
        }
    }
```

如果在创建ArrayList对象的时候就可以知道数组长度大概范围，通过ArrayList对象调用ensureCapacity()方法可以达到性能优化的效果。因为如果一开始没有设置容量大小，每当数据添加到数组中，数组容量不足就会动态的扩容，直接将数组元素复制到新的数组，新的数组容量大小为原来的1.5倍，这是非常耗性能的，假设原本的ArrayList对象已经有几十万条的数据，复制到新的数组等于将这几十万条数据的内存复制一遍，容易发生oom。因此如果在使用数组的时候就知道数据量的大小，我们应该先设置容量。

举个例子：

```java
public class Sample {
    public static void main(String[] args) {
        System.out.println("不设置集合容量大小，使用默认的容量大小");
        ArrayList arrayList  = new ArrayList();
        arrayList.add("1");
        System.out.println("实际集合容量大小为：" + getArrayListCapacity(arrayList));
        System.out.println("设置容量大小为5");
        arrayList.ensureCapacity(5);
        System.out.println("实际集合容量大小为：" + getArrayListCapacity(arrayList));
        System.out.println("设置容量大小为20");
        arrayList.ensureCapacity(20);
        System.out.println("实际集合容量大小为：" + getArrayListCapacity(arrayList));
    }

    // 通过反射获取私有变量--elementData数组的长度
    public static int getArrayListCapacity(ArrayList<?> arrayList) {
        Class<ArrayList> arrayListClass = ArrayList.class;
        try {
            Field field = arrayListClass.getDeclaredField("elementData");
            field.setAccessible(true);
            Object[] objects = (Object[])field.get(arrayList);
            return objects.length;
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
            return -1;
        } catch (IllegalAccessException e) {
            e.printStackTrace();
            return -1;
        }
    }
}
```

输出结果：
```java
不设置集合容量大小，使用默认的容量大小
实际集合容量大小为：10
设置容量大小为5
实际集合容量大小为：10
设置容量大小为20
实际集合容量大小为：20
```

#### 8.获取数组元素对应的索引——indexOf(Object o)、indexOf(Object o)

```java
    // 返回第一次出现元素的索引，如果没有则返回-1
    public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }

    // 返回最后一次出现元素的索引，如果没有则返回-1
    public int lastIndexOf(Object o) {
        if (o == null) { 
            for (int i = size-1; i >= 0; i--)
                if (elementData[i]==null)
                    return i;
        } else { 
            for (int i = size-1; i >= 0; i--)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```
根据源码可以看出，获取元素索引的方法很简单，就是遍历数组，直到遇到第一个或最后一个符合条件的元素，返回索引，如果都没有则返回-1。

示例：
```java
public class Sample {
    public static void main(String[] args) {
        ArrayList list = new ArrayList();
        list.add("A");
        list.add("B");
        list.add("C");
        list.add("D");
        list.add("E");
        list.add("A");
        list.add("B");
        System.out.println(list.indexOf("B"));
        System.out.println(list.lastIndexOf("B"));
    }
}
```
输出结果：
```java
1
6
```


#### 9.查看是否包含元素——contains(Object o)

```java
    // 是否包含元素
    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }
```
源码的实现非常简单，就是通过调用indexOf()的返回值判断是否包含元素，如果返回-1一定就是不包含元素，contains()方法就会返回false，否则返回true。

#### 10.获取集合长度——size()
```java
    public int size() {
        return size;// size属性的大小是随着集合添加或者删除元素变化
    }
```

#### 11.ArrayList集合克隆——clone()
```java
   // 重写Object父类的方法
   public Object clone() {
        try {
            ArrayList<?> v = (ArrayList<?>) super.clone();
            v.elementData = Arrays.copyOf(elementData, size);
            v.modCount = 0;
            return v;
        } catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError(e);
        }
    }
```

但有一点需要强调，ArrayList的clone()方法是浅克隆，并非深克隆。

什么是浅克隆？什么是深克隆？在原型模式中也有相关的概念【[鄙人的拙作](https://www.jianshu.com/p/dac220d3d314)】
>**浅克隆**：创建一个新对象，新对象的属性和原来对象完全相同，对于非基本类型属性，仍指向原有属性所指向的对象的内存地址。
**深克隆**：创建一个新对象，属性中引用的其他对象也会被克隆，不再指向原有对象地址。

示例说明：
```java
public class Sample {
    public static void main(String[] args) {
        Student student = new Student();
        student.setName("Muggle");

        ArrayList<Student> list = new ArrayList();
        list.add(student);

        // 浅克隆
        ArrayList<Student> cloneList = (ArrayList) list.clone();

        System.out.println("比较被克隆的对象和克隆对象是否一样：" + (list == cloneList));

        // 修改集合数组对象的元素
        student.setName("MuggleLee");

        System.out.println("被克隆集合的数组第一个元素：" + list.get(0).getName());
        System.out.println("克隆集合的数组第一个元素：" + cloneList.get(0).getName());
    }
}
class Student{
    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```
输出结果：
```java
比较被克隆的对象和克隆对象是否一样：false
被克隆集合的数组第一个元素：MuggleLee
克隆集合的数组第一个元素：MuggleLee
```

由输出结果可知，被克隆的集合与克隆的集合是不一样的，但是修改元素却都会影响两个集合。

关于被克隆集合、克隆集合和它们元素的关系如下：

![Clone-sample](https://raw.githubusercontent.com/MuggleLee/PicGo/master/ArrayList/clone-sample.png)

#### 12.集合元素转变成数组——toArray()、toArray(T[] a)

```java
public Object[] toArray() {
    return Arrays.copyOf(elementData, size);
}

@SuppressWarnings("unchecked")
public <T> T[] toArray(T[] a) {
    if (a.length < size)
        return (T[]) Arrays.copyOf(elementData, size, a.getClass());
    System.arraycopy(elementData, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}
```

源码很简单，两个方法其实都是调用System.arraycopy()方法将集合内部的数组内存复制到一个新的数组返回。不过要注意的是toArray()方法，因为容易抛出java.lang.ClassCastException的异常！

先示范代码：
```java
public class Sample {
    public static void main(String[] args) {
	ArrayList list = new ArrayList();
        list.add(1);
        list.add(2);
        list.add(3);
        Integer[] integers = (Integer[]) list.toArray();
    }
}
```
输出结果报的异常是：
> Exception in thread "main" java.lang.ClassCastException: [Ljava.lang.Object; cannot be cast to [Ljava.lang.Integer;


**这是因为toArray()返回数组的类型是Object[]类型，如果想通过强转类型为Integer[]或者其他数组属于向下转型，这在Java中是不允许的。**

那有什么办法可以转化为其他类型的数组呢？可以使用另外一种方法 toArray(T[] a) 。通过源码可以看出，toArray(T[] a)方法可以转变为任意类型的数组。
```java
public class Sample {
    public static void main(String[] args) {
	ArrayList list = new ArrayList();
        list.add(1);
        list.add(2);
        list.add(3);

	// 推荐使用的方式。因为新建的数组为0，复制出来的数组长度与集合数组长度一样，不会造成内存浪费
        Integer[] integers = (Integer[]) list.toArray(new Integer[0]);
        System.out.println(integers.length);// 输出结果为3
    }
}
```
#### 13.清空集合数组元素——clear()
```java
public void clear() {
    modCount++;

    // clear to let GC do its work
    for (int i = 0; i < size; i++)
        elementData[i] = null;

    size = 0;
}
```
源码很简单，将集合数组元素全部设置为null，便于垃圾回收。

#### 14.与Collection对象找交集——retainAll(Collection<?> c)

```java
    public boolean retainAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, true);
    }

    private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;
        int r = 0, w = 0;
        boolean modified = false;
        try {
            for (; r < size; r++)
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
        } finally {
            // Preserve behavioral compatibility with AbstractCollection,
            // even if c.contains() throws.
            if (r != size) {
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                w += size - r;
            }
            if (w != size) {
                // clear to let GC do its work
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


但有一点需要注意，我们不能够通过retainAll(Collection<?> c)的返回值判断两个集合是否有交集。为什么呢？

示例：
```java
public class Sample {
    public static void main(String[] args) {
        // 示例一
        ArrayList list1 = new ArrayList();
        list1.add(1);
        list1.add(2);

        LinkedList linkedList1 = new LinkedList();
        linkedList1.add(1);
        linkedList1.add(5);

        // 找出与linkedList交集
        boolean flag = list1.retainAll(linkedList1);
        System.out.println("示例一：执行retainAll()后的boolean值：" + flag);

        System.out.print("两个集合的交集为：");
        for (int i = 0; i < list1.size(); i++) {
            System.out.print("  " + list1.get(i));
        }
        System.out.println();

        // 示例二
        ArrayList list2 = new ArrayList();
        list2.add(1);
        list2.add(2);

        LinkedList linkedList2 = new LinkedList();
        linkedList2.add(1);
        linkedList2.add(2);
        linkedList2.add(3);

        boolean flag2 = list2.retainAll(linkedList2);
        System.out.println("示例二：执行retainAll()后的boolean值：" + flag2);

        System.out.print("两个集合的交集为：");
        for (int i = 0; i < list2.size(); i++) {
            System.out.print("    " + list2.get(i));
        }
    }
}
```
输出结果：
```java
示例一：执行retainAll()后的boolean值：true
两个集合的交集为：  1
示例二：执行retainAll()后的boolean值：false
两个集合的交集为：    1    2
```
两个集合的关系图如下：

![Collection-relation](https://raw.githubusercontent.com/MuggleLee/PicGo/master/ArrayList/Collection-relation.png)

示例一和示例二中两个集合都有交集，但为什么示例二中的retainAll(Collection<?> c)返回值为false？其实retainAll(Collection<?> c)方法的作用是判断调用者是否有元素被移除。如果调用者的元素没有被移除证明 c 集合的元素包含调用者的全部元素。

#### 15.ArrayList集合排序——sort(Comparator<? super E> c)
```java
    public void sort(Comparator<? super E> c) {
        final int expectedModCount = modCount;
        Arrays.sort((E[]) elementData, 0, size, c);
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
        modCount++;
    }
```

根据源码可以知道，使用sort()方法的时候需要传入一个Comparator的对象，那该怎么使用呢？

示例：
```java
public class Sample {
    public static void main(String[] args) {
        ArrayList<Integer> list = new ArrayList();
        list.add(1);
        list.add(3);
        list.add(5);
        list.add(4);
        list.add(2);

        // 第一种方法：实现Comparator接口的compare方法
        list.sort(new Comparator<Integer>() {
            @Override
            public int compare(Integer o1, Integer o2) {
                if(o1 > o2){
                    return 1;
                }else {
                    return -1;
                }
            }
        });

        // 第二种方式：使用lambda表达式
        //list.sort((x, y) -> x > y ? 1 : -1);

        // 第三种方式：使用JDK1.8中Comparator接口提供的排序方法。正叙：naturalOrder() 倒叙：reverseOrder()
        //list.sort(Comparator.naturalOrder());

        // 第四种方式：甚至可以不用传Comparator对象，直接传null也可以排序
        //list.sort(null);

        for (int i = 0; i < list.size(); i++) {
            System.out.print(list.get(i) + "    ");
        }
    }
}
```
输出结果：
```java
1    2    3    4    5    
```
使用ArrayList.sort()方法底层都是调用Arrays.sort()方法【】



#### 16.多线程遍历集合——spliterator() 

这个高大上的方法可以在集合数据量大的时候结合多线程遍历元素。先通过简单的例子了解是如何使用。
```java
public class SpliteratorDemo {
    public static void main(String[] args) {
        ArrayList<String> list = new ArrayList<>();
        list.add("1");
        list.add("2");
        list.add("3");
        list.add("4");
        list.add("5");
        list.add("6");
        list.add("7");
        list.add("8");
        Spliterator spliterator1 = list.spliterator();
        Spliterator spliterator2 = spliterator1.trySplit();
        Spliterator spliterator3 = spliterator1.trySplit();
        Spliterator spliterator4 = spliterator2.trySplit();
        System.out.println("spliterator1的长度为：" + spliterator1.estimateSize());
        System.out.println("spliterator2的长度为：" + spliterator2.estimateSize());
        System.out.println("spliterator3的长度为：" + spliterator3.estimateSize());
        System.out.println("spliterator4的长度为：" + spliterator4.estimateSize());
        spliterator1.forEachRemaining(x -> System.out.println("spliterator1对象："+x));
        spliterator2.forEachRemaining(x -> System.out.println("spliterator2对象："+x));
        spliterator3.forEachRemaining(x -> System.out.println("spliterator3对象："+x));
        spliterator4.forEachRemaining(x -> System.out.println("spliterator4对象："+x));

    }
}

```


输出结果：
```java
spliterator1的长度为：2
spliterator2的长度为：2
spliterator3的长度为：2
spliterator4的长度为：2
spliterator1对象：7
spliterator1对象：8
spliterator2对象：3
spliterator2对象：4
spliterator3对象：5
spliterator3对象：6
spliterator4对象：1
spliterator4对象：2
```
由输出结果可以知道，通过list.spliterator()创建Spliteror对象，再调用trySplit()方法可以将集合数组平均分配。例子中将集合元素平均分配给4个Spliteror对象，调用forEachRemaining()方法可以遍历循环集合元素。

在实际开发中，如果集合元素比较多，可以考虑使用Spliteror对象结合多线程并发遍历元素，可以提高效率！不过值得注意的是，使用多线程不一定会提高执行效率的，如果多线程执行的运算不高，CPU算法调度线程有可能将所有线程都分配在一个内核执行，实际上就是单核执行多线程，加上线程之间的上下文切换等时间，多线程的使用反而是降低系统运行效率的！【[可以参考我的另外一篇文章](https://www.jianshu.com/p/352caffd6366)】


#### 17.使用forEach(Consumer<? super E> action)迭代集合

```java
    public void forEach(Consumer<? super E> action) {
        Objects.requireNonNull(action);// 检查Comsumer对象是否为null
        final int expectedModCount = modCount;
        @SuppressWarnings("unchecked")
        final E[] elementData = (E[]) this.elementData;// 获取集合的数组
        final int size = this.size;// 数组长度
        for (int i=0; modCount == expectedModCount && i < size; i++) {
            action.accept(elementData[i]);// 数组元素作为accept的参数执行accept方法
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }
```



##### *forEach(Consumer<? super E> action)的行为取决于accept()的实现

举个例子：

```java
public class Sample {
    public static void main(String[] args) {
        ArrayList<Integer> list = new ArrayList();
        list.add(1);
        list.add(3);
        list.add(5);
        list.add(4);
        list.add(2);

        // 第一种方式：实现Consumer接口的accept方法
//        list.forEach(new Consumer<Integer>() {
//            @Override
//            public void accept(Integer integer) {
//                System.out.println(integer);
//            }
//        });

        // 第二种方式：使用lambda表达式
        list.forEach(x -> System.out.print(x + " "));

        System.out.println();

        // 执行forEach()方法，将数组元素的值都扩大十倍
        list.forEach(x -> System.out.print(x * 10 + " "));
    }
}
```
输出结果：
```java
1 3 5 4 2 
10 30 50 40 20 
```


#### 18.移除符合条件的集合元素——removeIf()

```java
    public boolean removeIf(Predicate<? super E> filter) {
        Objects.requireNonNull(filter);// 判断Predicate对象是否为null
        int removeCount = 0;
        final BitSet removeSet = new BitSet(size);// 可以简单理解为一个set集合，作用是记录使test()方法返回true的元素索引
        final int expectedModCount = modCount;
        final int size = this.size;
        // 将test()方法返回true的元素索引set进removeSet里面
        for (int i=0; modCount == expectedModCount && i < size; i++) {
            @SuppressWarnings("unchecked")
            final E element = (E) elementData[i];
            if (filter.test(element)) {
                removeSet.set(i);
                removeCount++;
            }
        }
        // 并发情况下可能会不相同，则抛出异常
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }

        // 如果removeCount大于0代表有集合元素使test()方法返回true，并将集合数组中使test()方法返回true的元素索引移除
        final boolean anyToRemove = removeCount > 0;
        if (anyToRemove) {
            final int newSize = size - removeCount;
            for (int i=0, j=0; (i < size) && (j < newSize); i++, j++) {
                i = removeSet.nextClearBit(i);
                elementData[j] = elementData[i];
            }
            for (int k=newSize; k < size; k++) {
                elementData[k] = null;  // Let gc do its work
            }
            this.size = newSize;
            if (modCount != expectedModCount) {
                throw new ConcurrentModificationException();
            }
            modCount++;
        }
        return anyToRemove;
    }
```
举个例子：
```java
public class Sample {
    public static void main(String[] args) {
        ArrayList<Integer> list = new ArrayList();
        list.add(1);
        list.add(3);
        list.add(5);

        // 第一种方式：实现Predicate接口的test方法
//        list.removeIf(new Predicate<Integer>() {
//            @Override
//            public boolean test(Integer integer) {
//                return integer == 1;
//            }
//        });

        // 使用lambda表达式
        list.removeIf(x -> x == 1);

        list.forEach(x -> System.out.print(x + " "));
    }
}
```
输出结果：
```java
3 5
```


#### 19.返回集合区间内的所有元素——subList(int fromIndex, int toIndex)

```java
    public List<E> subList(int fromIndex, int toIndex) {
        subListRangeCheck(fromIndex, toIndex, size);// 检查索引值的范围，如果索引值的范围有错直接抛异常
        return new SubList(this, 0, fromIndex, toIndex);// 返回一个SubList的对象，这个对象与List的对象具有相同的实现，故不再赘述。
    }
```
举个例子：
```java
public class Sample {
    public static void main(String[] args) {
        ArrayList<Integer> list = new ArrayList();
        list.add(1);
        list.add(3);
        list.add(5);
        list.add(4);

        // 将list集合数组索引为1，2区间的元素赋值给新的集合，值得注意的是，执行subList()方法不会对原集合有影响
        List subList = list.subList(1,2);
        System.out.println("原集合：");
        list.forEach(x-> System.out.print(x + " "));
        System.out.println();
        System.out.println("新的集合：");
        subList.forEach(x-> System.out.print(x + " "));
    }
}
```
输出结果：
```java
原集合：
1 3 5 4 
新的集合：
3 
```


参考资料：

[https://www.jianshu.com/p/dac220d3d314](https://www.jianshu.com/p/dac220d3d314)


