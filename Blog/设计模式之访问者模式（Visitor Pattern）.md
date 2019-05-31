

## What:

>提供一个作用于某对象结构中的各元素的操作表示，它使得可以在不改变各元素的类的前提下定义作用于这些元素的新操作。


## Why:
#### 优点：


#### 缺点：


## Where:


## How:

访问者模式有以下几个角色：

**抽象访问者（Visitor）角色：** 定义一个访问具体元素的接口，为每个具体元素类对应一个访问操作 visit() ，该操作中的参数类型标识了被访问的具体元素。

**具体访问者（ConcreteVisitor）角色：** 实现抽象访问者角色中声明的各个访问操作，确定访问者访问一个元素时该做什么。

**抽象元素（Element）角色：** 声明一个包含接受操作 accept() 的接口，被接受的访问者对象作为 accept() 方法的参数。

**具体元素（ConcreteElement）角色：** 实现抽象元素角色提供的 accept() 操作，其方法体通常都是 visitor.visit(this) ，另外具体元素中可能还包含本身业务逻辑的相关操作。

**对象结构（Object Structure）角色：** 是一个包含元素角色的容器，提供让访问者对象遍历容器中的所有元素的方法，通常由 List、Set、Map 等聚合类实现。

Staff（抽象元素）：
```java
public interface Staff {
    void accept(Department department);
}
```
Manager、Engineer（具体元素）：
```java
public class Engineer implements Staff {

    private String staffName;

    private double workTime;

    public Engineer(String staffName) {
        this.staffName = staffName;
    }


    @Override
    public void accept(Department department) {
        department.visit(this);
    }

    /
}
```






Test:测试类
```java

```
输出结果：
```java

```


# 总结

参考资料:
