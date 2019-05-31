

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

    //省略get、set方法
}

public class Manager implements Staff {

    private String staffName;

    private double workTime;

    public Manager(String staffName) {
        this.staffName = staffName;
    }


    @Override
    public void accept(Department department) {
        department.visit(this);
    }

    //省略get、set方法
}
```
Department（抽象访问者）：
```java
public interface Department {
    void visit(Engineer engineer);

    void visit(Manager manager);
}
```
AccountingDepartment、PersonnelDepartment（具体访问者）：

```java
public class AccountingDepartment implements Department{
    private final double MANAGER_DAILY_SALARY = 1000D;

    private final double ENGINEER_DAILY_SALARY = 500D;

    @Override
    public void visit(Engineer engineer) {
        System.out.println(engineer.getStaffName() + " 工程师，月薪为：" + engineer.getWorkTime() * ENGINEER_DAILY_SALARY);
    }

    @Override
    public void visit(Manager manager) {
        System.out.println(manager.getStaffName() + " 经理，月薪为：" + manager.getWorkTime() * MANAGER_DAILY_SALARY);
    }
}
public class PersonnelDepartment implements Department {
    @Override
    public void visit(Engineer engineer) {
        engineer.setWorkTime(workTime());
        System.out.println(engineer.getStaffName() + " 工程师，上个月工作天数为：" + workTime());
    }

    @Override
    public void visit(Manager manager) {
        manager.setWorkTime(workTime());
        System.out.println(manager.getStaffName() + " 经理，上个月工作天数为：" + workTime());
    }

    //默认所有人都工作22天
    double workTime(){
        return 22;
    }
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
