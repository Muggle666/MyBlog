之前写了几篇关于基本数据类型的原子类、数组类型的原子类和引用对象类型的原子类，重点介绍了AtomicInteger、AtmoicIntegerArray和AtomicStampedReference原子类。而接下来这篇文章，重点AtomicIntegerFieldUpdater原子类！

# 简介


**AtomicIntegerFieldUpdater**原子类字面意思可以理解为：整型字段原子更新器。可以原子地修改对象的属性，与其类似的还有**AtomicLongFieldUpdater**和**AtomicReferenceFieldUpdater**。

#### 思考：
>AtomicIntegerFieldUpdater和AtomicInteger有什么区别呢？AtomicInteger也可以原子化地修改对象的属性呀，那AtomicIntegerFieldUpdater的应用场景是什么？

首先回顾一下AtomicInteger原子类的使用，当我们声明一个变量为AtomicInteger原子类，累加操作等算法操作都是调用getXXX()、setXXX()之类的方法。

```java
AtomicInteger x = new AtomicInteger(1);
x.incrementAndGet();
x.set(10);
```


那如果在原有的代码中，一个对象的属性 x 只是用基本数据类型或者引用类型声明，通过getter、setter等方式修改变量的值，在并发情况下无法保证原子化，有可能导致数据异常。此时如果变量 x
 由int类型变成AtomicInteger原子类声明，虽然可以保证原子性，但在每一处使用到这个变量 x 的地方都要修改为AtomicInteger原子类相应的方法，这很明显违背了设计模式中的六大原则——“开闭原则”！

```java
//原有的代码
int x = 0;
public void setX(){...}
public int getX(){...}

//为了保证原子性，修改后的代码
AtomicInteger x = new AtomicInteger(0);
x.set(...);
x.get();
```

所以此时使用AtomicIntegerFieldUpdater原子类就大派用场啦！我认为使用AtomicIntegerFieldUpdater原子类的使用场景最主要的就是可以不用修改过多的代码就可以保证代码的原子性操作！


### AtomicIntegerFieldUpdater原子类的使用

使用AtomicIntegerFieldUpdater原子类有三点注意事项：
1.变量必须使用volatile关键字修饰。
>使用volatile是为了保证可见性，如果没有volatile关键字修饰，使用newUpdater()会抛出IllegalArgumentException异常。

2.不能使用static关键字。
>

3.不能使用final关键字。
>这个容易理解，因为需要修改变量嘛，设置final就不能修改变量啦。

4.变量的描述符类型必须与调用者一致。如果调用者能够调用变量就能够通过反射操作保证原子性。


示例：
```java
1. public class AtomicIntegerFieldUpdaterDemo {
2. 
3.     //设置100000个线程，模拟并发场景
4.     private static final int THREAD_NUM = 100000;
5. 
6.     //设置栅栏是为了防止循环还没结束就执行main线程输出自增的变量，导致误以为线程不安全
7.     private static CountDownLatch countDownLatch = new CountDownLatch(THREAD_NUM);
8. 
9.     private static AtomicIntegerFieldUpdater atomicIntegerFieldUpdater = AtomicIntegerFieldUpdater.newUpdater(Score.class, "totalScore");
10. 
11.     public static void main(String[] args) throws InterruptedException {
12.         Score score = new Score();
13.         for (int j = 0; j < THREAD_NUM; j++) {
14.             new Thread(() -> {
15.                 atomicIntegerFieldUpdater.incrementAndGet(score);
16.                 countDownLatch.countDown();
17.             }).start();
18.         }
19.         countDownLatch.await();
20.         System.out.println("totalScore的值：" + atomicIntegerFieldUpdater.get(score));
21.     }
22. }
23. 
24. class Score {
25.     public volatile int totalScore = 0;
26. 
27.     public int getTotalScore() {
28.         return totalScore;
29.     }
30. 
31.     public void setTotalScore(int totalScore) {
32.         this.totalScore = totalScore;
33.     }
34. }
```
输出结果：
```java
totalScore的值：100000
```

第9行AtomicIntegerFieldUpdater调用newUpdater(Class&lt;U&gt tclass, String fieldName)静态方法创建AtomicIntegerFieldUpdater对象，通过AtomicIntegerFieldUpdater的对象就可以保证对象属性为参数fieldName的原子性，示例中为Score类中的totalScore属性。



先看下AtomicIntegerFieldUpdater原子类有哪些方法：

![AtomicIntegerFieldUpdater类图](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Atomic/AtomicIntegerFieldUpdater/AtomicIntegerFieldUpdater-UML.png)


咦？为什么AtomicIntegerFieldUpdater原子类内部又有AtomicIntegerFieldUpdaterImpl私有类，并且继承AtomicIntegerFieldUpdater类？原来AtomicIntegerFieldUpdater是一个抽象类，内部通过AtomicIntegerFieldUpdaterImpl类实现父类。
```java
public abstract class AtomicIntegerFieldUpdater<T> {...}
private static final class AtomicIntegerFieldUpdaterImpl<T> extends AtomicIntegerFieldUpdater<T> {...}
```

AtomicIntegerFieldUpdater原子类的构造函数修饰符为protect，提供一个静态方法newUpdater()创建AtomicIntegerFieldUpdater的对象。

```java
    @CallerSensitive
    public static <U> AtomicIntegerFieldUpdater<U> newUpdater(Class<U> tclass,
                                                              String fieldName) {
        return new AtomicIntegerFieldUpdaterImpl<U>
            (tclass, fieldName, Reflection.getCallerClass());
    }
```

**@CallerSensitive** 这个是JVM的注释，但目前我还找不到明确的解释，但兴趣的朋友可以参考这位大佬写的博客【[JVM注解@CallerSensitive](https://blog.csdn.net/HEL_WOR/article/details/50199797)】

通过这个静态方法调用子类AtomicIntegerFieldUpdaterImpl的构造函数。
```java
private static final class AtomicIntegerFieldUpdaterImpl<T>
        extends AtomicIntegerFieldUpdater<T> {
    // Unsafe对象
    private static final sun.misc.Unsafe U = sun.misc.Unsafe.getUnsafe();
    // 变量的内存偏移量
    private final long offset;
    // 如果字段受保护，cclass为调用者类的class对象，否则cclass为tclass。
    private final Class<?> cclass;
    // 被操作对象类的class对象
    private final Class<T> tclass;

    /**
     * @param tclass 被操作对象类的class对象（相当于上面例子的Score.class）
     * @param fieldName 被操作对象的属性（相当于上面例子Score类的totalScore属性）
     * @param caller 调用者类的class对象（相当于上面例子的AtomicIntegerFieldUpdaterDemo.class）
     */
    AtomicIntegerFieldUpdaterImpl(final Class<T> tclass,
                                  final String fieldName,
                                  final Class<?> caller) {
        final Field field; // 要原子更新的字段
        final int modifiers; // 字段的修饰符
        try {
            // 通过反射获得tclass的fieldName
            field = AccessController.doPrivileged(
                    new PrivilegedExceptionAction<Field>() {
                        public Field run() throws NoSuchFieldException {
                            return tclass.getDeclaredField(fieldName);
                        }
                    });
            // 获取字段的修饰符
            modifiers = field.getModifiers();
            // 检查字段的访问权限，如果不在访问范围内则抛出异常
            sun.reflect.misc.ReflectUtil.ensureMemberAccess(caller, tclass, null, modifiers);
            // 获取对象的类装载器
            ClassLoader cl = tclass.getClassLoader();
            ClassLoader ccl = caller.getClassLoader();
            if ((ccl != null) && (ccl != cl) &&
                    ((cl == null) || !isAncestor(cl, ccl))) {
                sun.reflect.misc.ReflectUtil.checkPackageAccess(tclass);
            }
        } catch (PrivilegedActionException pae) {
            throw new RuntimeException(pae.getException());
        } catch (Exception ex) {
            throw new RuntimeException(ex);
        }

        // 如果属性的类型不是int基本数据类型则抛出异常
        if (field.getType() != int.class)
            throw new IllegalArgumentException("Must be integer type");

        // 如果字段的修饰符不是volatile则抛出异常
        if (!Modifier.isVolatile(modifiers))
            throw new IllegalArgumentException("Must be volatile type");


        this.cclass = (Modifier.isProtected(modifiers) && // 字段的修饰符是否是protect
                tclass.isAssignableFrom(caller) && // 判断tclass和caller是否相同或者是另一个类的子类或接口
                !isSamePackage(tclass, caller)) // 是否具有相同的ClassLoader和包名称
                ? caller : tclass;
        this.tclass = tclass;
        // 获取该字段在对象内存中的偏移量
        this.offset = U.objectFieldOffset(field);
    }

    // second是否在first的类加载器委托链上（双亲委派模型）
    private static boolean isAncestor(ClassLoader first, ClassLoader second) {
        ClassLoader acl = first;
        do {
            acl = acl.getParent();
            if (second == acl) {
                return true;
            }
        } while (acl != null);
        return false;
    }

    // 是否具有相同的ClassLoader和包名称
    private static boolean isSamePackage(Class<?> class1, Class<?> class2) {
        return class1.getClassLoader() == class2.getClassLoader()
                && Objects.equals(getPackageName(class1), getPackageName(class2));
    }

    // 获取包名称
    private static String getPackageName(Class<?> cls) {
        String cn = cls.getName();
        int dot = cn.lastIndexOf('.');
        return (dot != -1) ? cn.substring(0, dot) : "";
    }
}
```

通过AtomicIntegerFieldUpdater.newUpdater(...)创建到AtomicIntegerFieldUpdater对象之后，就可以像使用AtomicInteger原子类一样，方法基本相似，只不过有一处地方比较特殊。就以上面例子中的incrementAndGet(T obj)为例吧！

```java
public final int incrementAndGet(T obj) {
    return getAndAdd(obj, 1) + 1;
}
public final int getAndAdd(T obj, int delta) {
    accessCheck(obj);
    return U.getAndAddInt(obj, offset, delta);
}
// obj是否是cclass的实例
private final void accessCheck(T obj) {
    if (!cclass.isInstance(obj))
    	throwAccessCheckException(obj);
}
```

使用incrementAndGet()方法的时候，需要传入原子性操作属性的对象，例子中就是Score的对象，可以理解为“通知”AtomicIntegerFieldUpdater对象我需要保证哪个对象属性的原子性。而在操作CAS之前需要执行accessCheck(T obj)的方法，该方法检查参数传入的对象是否是cclass的实例。为什么比AtomicInteger原子类操作CAS方法多一个检查的步骤呢？譬如我需要A类中的属性原子化，但我操作的时候粗心大意传入B类的对象，那很明显是错误的代码，所以需要检查是否为同一个实例！

AtomicIntegerFieldUpdater原子类中还有大量其它的CAS方法，但与AtomicInteger原子类基本相似，故不再赘述。

### AtomicLongFieldUpdater原子类的使用

与AtomicIntegerFieldUpdater相似，AtomicLongFieldUpdater也是一个抽象类，但有两个内部类分别继承AtomicLongFieldUpdater抽象类。

先来看下AtomicLongFieldUpdater抽象类的类图。
![AtomicLongFieldUpdater类图](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Atomic/AtomicLongFieldUpdater/AtomicLongFieldUpdater-UML.jpg)

在执行AtomicLongFieldUpdater.newUpdater()方法的时候选择使用哪一个内部类，那为什么需要有两个内部类呢？带着疑问，打开AtomicLongFieldUpdater.newUpdater()的源码！

```java
@CallerSensitive
public static <U> AtomicLongFieldUpdater<U> newUpdater(Class<U> tclass, String fieldName) {
    Class<?> caller = Reflection.getCallerClass();
    if (AtomicLong.VM_SUPPORTS_LONG_CAS) // JVM支持lockless的 CAS 操作
        return new CASUpdater<U>(tclass, fieldName, caller);
    else // JVM不支持lockless的 CAS 操作
        return new LockedUpdater<U>(tclass, fieldName, caller);
}
```

newUpdater()方法内通过AtomicLong.VM_SUPPORTS_LONG_CAS判断JVM是否支持lockless的CAS操作。
###### 关于AtomicLong的解释，可以浏览本人的拙作【原子类系列之AtomicInteger】。

接下来继续打开CASUpdater和LockedUpdater内部类的构造器源码：

```java
    CASUpdater(final Class<T> tclass, final String fieldName, final Class<?> caller) {
        final Field field;
        final int modifiers;
        try {
            field = AccessController.doPrivileged(
                    new PrivilegedExceptionAction<Field>() {
                        public Field run() throws NoSuchFieldException {
                            return tclass.getDeclaredField(fieldName);
                        }
                    });
            modifiers = field.getModifiers();
            sun.reflect.misc.ReflectUtil.ensureMemberAccess(
                    caller, tclass, null, modifiers);
            ClassLoader cl = tclass.getClassLoader();
            ClassLoader ccl = caller.getClassLoader();
            if ((ccl != null) && (ccl != cl) &&
                    ((cl == null) || !isAncestor(cl, ccl))) {
                sun.reflect.misc.ReflectUtil.checkPackageAccess(tclass);
            }
        } catch (PrivilegedActionException pae) {
            throw new RuntimeException(pae.getException());
        } catch (Exception ex) {
            throw new RuntimeException(ex);
        }

        if (field.getType() != long.class)
            throw new IllegalArgumentException("Must be long type");

        if (!Modifier.isVolatile(modifiers))
            throw new IllegalArgumentException("Must be volatile type");

        this.cclass = (Modifier.isProtected(modifiers) &&
                tclass.isAssignableFrom(caller) &&
                !isSamePackage(tclass, caller))
                ? caller : tclass;
        this.tclass = tclass;
        this.offset = U.objectFieldOffset(field);
    }

    LockedUpdater(final Class<T> tclass, final String fieldName, final Class<?> caller) {
        Field field = null;
        int modifiers = 0;
        try {
            field = AccessController.doPrivileged(
                    new PrivilegedExceptionAction<Field>() {
                        public Field run() throws NoSuchFieldException {
                            return tclass.getDeclaredField(fieldName);
                        }
                    });
            modifiers = field.getModifiers();
            sun.reflect.misc.ReflectUtil.ensureMemberAccess(
                    caller, tclass, null, modifiers);
            ClassLoader cl = tclass.getClassLoader();
            ClassLoader ccl = caller.getClassLoader();
            if ((ccl != null) && (ccl != cl) &&
                    ((cl == null) || !isAncestor(cl, ccl))) {
                sun.reflect.misc.ReflectUtil.checkPackageAccess(tclass);
            }
        } catch (PrivilegedActionException pae) {
            throw new RuntimeException(pae.getException());
        } catch (Exception ex) {
            throw new RuntimeException(ex);
        }

        if (field.getType() != long.class)
            throw new IllegalArgumentException("Must be long type");

        if (!Modifier.isVolatile(modifiers))
            throw new IllegalArgumentException("Must be volatile type");

        this.cclass = (Modifier.isProtected(modifiers) &&
                tclass.isAssignableFrom(caller) &&
                !isSamePackage(tclass, caller))
                ? caller : tclass;
        this.tclass = tclass;
        this.offset = U.objectFieldOffset(field);
    }
```

不难发现，CASUpdater和LockedUpdater构造器的源码与AtomicIntegerFieldUpdaterImpl构造器的差不多一模一样，所以不作过多的解释。值得注意的是，LockedUpdater内部类的方法都是使用synchronized关键字保证线程安全的，这是在JVM不支持lockless的CAS操作情况下使用的，性能相对于CASUpdater会低一点。

### AtomicReferenceFieldUpdater原子类的使用

除了基本数据类型的原子更新器，还有一个原子类是对象的原子更新器——AtomicReferenceFieldUpdater。

AtomicReferenceFieldUpdater原子类与AtomicIntegerFieldUpdater和AtomicLongFieldUpdater原子类的使用基本相似，源码不再赘述。

写个简单例子使用AtomicReferenceFieldUpdater
```java
public class AtomicReferenceFieldUpdaterDemo {

    public static AtomicReferenceFieldUpdater updater = AtomicReferenceFieldUpdater.newUpdater(Student.class, Integer.class, "score");
    public static Student student = new Student();

    public static void main(String[] args) {
        System.out.println("获取指定对象的指定属性值：" + updater.get(student));

        updater.compareAndSet(student,0,10);
        System.out.println("执行compareAndSet(student,0,10)后指定对象的指定属性值：" + updater.get(student));

        updater.set(student,20);
        System.out.println("执行set(student,20)后指定对象的指定属性值：" + updater.get(student));
        
//        updater.updateAndGet(student, new UnaryOperator() {
//            @Override
//            public Object apply(Object o) {
//                return 100;
//            }
//        });
        updater.updateAndGet(student,(s1)->(100));
        System.out.println("执行updateAndGet(student,(s1)->(100))后指定对象的指定属性值：" + updater.get(student));
    }
}

class Student {
    public volatile Integer score = 0;
}
```
输出结果：
```java
获取指定对象的指定属性值：0
执行compareAndSet(student,0,10)后指定对象的指定属性值：10
执行set(student,20)后指定对象的指定属性值：20
执行updateAndGet(student,(s1)->(100))后指定对象的指定属性值：100
```


# 总结

当需要保证对象的属性原子化操作，我们肯定优先考虑使用java.util.concurrent.atomic 包下提供的**字段原子更新器（AtomicIntegerFieldUpdater、AtomicLongFieldUpdater和AtomicReferenceFieldUpdater）**。这三个原子类都是抽象类，通过内部类继承并实现抽象方法。同时也了解到long和double类型在

参考资料：

[https://blog.csdn.net/HEL_WOR/article/details/50199797](https://blog.csdn.net/HEL_WOR/article/details/50199797)