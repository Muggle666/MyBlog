# 设计模式之迭代器模式
## What:
>提供一种方法顺序访问一个聚合对象中各个元素, 而又无须暴露该对象的内部表示。



## Why:
#### 优点：
1.不需要暴露内部表示就可以访问聚合对象；
2.支持不同方式遍历聚合对象；
3.迭代器简化了聚合类；
4.不需要改动原有的代码就可以很方便的增加新的聚合类和迭代器类，符合开闭原则。

#### 缺点：
由于迭代器模式将存储数据和遍历数据的职责分离，增加新的聚合类需要对应增加新的迭代器类，类的个数成对增加，这在一定程度上增加了系统的复杂性。

## Where:
1.当需要为聚合对象提供多种遍历方式时。
2.当需要为遍历不同的聚合结构提供一个统一的接口时。
3.当访问一个聚合对象的内容而无须暴露其内部细节的表示时。

## How:

迭代器模式包括以下几种角色：
**抽象聚合（Aggregate）角色：** 定义存储、添加、删除聚合对象以及创建迭代器对象的接口。
**具体聚合（ConcreteAggregate）角色：** 实现抽象聚合类，返回一个具体迭代器的实例。
**抽象迭代器（Iterator）角色：** 定义访问和遍历聚合元素的接口，通常包含 hasNext()、first()、next() 等方法。
**具体迭代器（Concretelterator）角色：** 实现抽象迭代器接口中所定义的方法，完成对聚合对象的遍历，记录遍历的当前位置。

![迭代器模式UML](https://raw.githubusercontent.com/MuggleLee/PicGo/master/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E8%BF%AD%E4%BB%A3%E5%99%A8%E6%A8%A1%E5%BC%8F/IteratorPattern.png)

示例：假设需要按顺序的输出公司名称和公司部门。但是公司名称是由集合的形式保存，而公司部门用数组的形式保存，为了不改动原有的类，又不想暴露内部细节（即不希望客户端知道是使用什么形式保存的数据）。

![IteratorPattern-Sample](https://raw.githubusercontent.com/MuggleLee/PicGo/master/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E8%BF%AD%E4%BB%A3%E5%99%A8%E6%A8%A1%E5%BC%8F/IteratorPattern-Sample.png)

**MyIterator（抽象迭代器）：**
```java
public interface MyIterator {
    Object next();

    boolean hasNext();
}
```
**ListMyIterator类、ArrayMyIterator类（具体迭代器）：**
```java
public class ListMyIterator implements MyIterator {

    private List list;

    private Integer index = -1;

    public ListMyIterator(List list) {
        this.list = list;
    }

    @Override
    public Object next() {
        if(this.hasNext()){
            return list.get(++index);
        }
        return null;
    }

    @Override
    public boolean hasNext() {
        if(index < list.size()-1){
            return true;
        }else{
            return false;
        }
    }
}

public class ArrayMyIterator implements MyIterator{

    Object[] obj ;

    private Integer index = -1;

    public ArrayMyIterator(Object[] obj) {
        this.obj = obj;
    }

    @Override
    public Object next() {
        Object object = obj[++index];
        if(object != null){
            return object;
        }
       return "";
    }

    @Override
    public boolean hasNext() {
        return index < obj.length-1;
    }
}
```
**Operation（抽象聚合）：**
```java
public interface Operation {

    void add(Object obj);

    MyIterator createIterator();
}
```

**Company类、Management类（具体聚合）：**
```java
public class Company implements Operation {

    private List list = new ArrayList();

    @Override
    public void add(Object obj) {
        list.add(obj);
    }

    @Override
    public MyIterator createIterator() {
        return new ListMyIterator(this.list);
    }
}

public class Management implements Operation {

    static final Integer MAX_LENGTH = 100;

    private Object[] array = new Object[MAX_LENGTH];

    private int index = -1;

    @Override
    public void add(Object obj) {
        ++index;
        if(index != MAX_LENGTH){
            array[index] = obj;
        }else {
            System.out.println("达到容器最大长度！");
        }
    }

    @Override
    public MyIterator createIterator() {
        return new ArrayMyIterator(array);
    }
}
```
Test:测试类
```java
public class Test {
    public static void main(String[] args) {
        Operation company = new Company();
        company.add("广州");
        company.add("北京");
        company.add("上海");
        company.add("杭州");
        MyIterator companyIterator = company.createIterator();
        while (companyIterator.hasNext()) {
            String obj = (String) companyIterator.next();
            System.out.println(obj);
        }
        System.out.println("************** 华丽的分割线 **************");
        Operation management = new Management();
        management.add("市场部");
        management.add("销售部");
        management.add("人事部");
        management.add("技术部");
        MyIterator managementIterator = management.createIterator();
        while (managementIterator.hasNext()) {
            String obj = (String) managementIterator.next();
            System.out.println(obj);
        }
    }
}
```
输出结果：
```java
广州
北京
上海
杭州
************** 华丽的分割线 **************
市场部
销售部
人事部
技术部
```

# 总结
由测试代码可知，客户端不需要知道Company类和Management类分别用什么形式存储数据，而是统一的使用相同代码就能够访问对象，遵循“迪米特法则”；由于需要遍历的对象类没有声明迭代器的实现，而是引用外部的迭代器类，职责分离，因此也遵循“单一职责原则”。

参考资料:
[https://www.runoob.com/design-pattern/iterator-pattern.html](https://www.runoob.com/design-pattern/iterator-pattern.html)
