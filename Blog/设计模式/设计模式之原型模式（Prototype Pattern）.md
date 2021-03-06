# 设计模式之原型模式
## What？
> 是用于创建重复的对象，同时又能保证性能的一种创建型模式。

##  Why
**优点：** 
1、性能提高。当创建新的对象比较复杂时，可以利用原型模式简化对象的创建过程，同时也能够提高效率。
> 因为通过clone创建的一个对象比直接用new创建对象更有效率。

 2、逃避构造函数的约束。

**缺点：**
1. 在实现深克隆的时候可能需要比较复杂的代码。
2. 需要为每一个类配备一个克隆方法，而且这个克隆方法需要对类的功能进行通盘考虑，这对全新的类来说不是很难，但对已有的类进行改造时，不一定是件容易的事，必须修改其源代码，违背了“开闭原则”。

## Where
1. 当通过new产生一个对象需要非常繁琐的数据准备或访问权限。
3. 一个对象多个修改者的场景。
4. 类初始化需要消化非常多的资源，这个资源包括数据、硬件资源等。
5. 资源优化场景。

## How
>1. 实现Cloneable接口
>2. 重写clone方法

示例：一个班级有多个学生，每个学生只对应一个班级。
学生类：
```java
public class Student implements Cloneable {
    private String name;
    private Integer age;
    private Classroom classroom;

    public Student() {
        System.out.println("Student类的无参构造器...");
    }

    @Override
    public String toString() {
        final StringBuilder sb = new StringBuilder("{");
        sb.append("\"Name\":\"")
                .append(name).append('\"');
        sb.append(",\"Age\":")
                .append(age);
        sb.append(",\"Classroom\":")
                .append(classroom);
        sb.append('}').append("    ").append(super.toString());
        return sb.toString();
    }

    //省略get、set方法

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}
```
课室类(Classroom)：
```java
public class Classroom{

    private String className;

    public Classroom() {
    }

    @Override
    public String toString() {
        final StringBuilder sb = new StringBuilder("{");
        sb.append("\"ClassName\":\"")
                .append(className).append('\"');
        sb.append('}').append(super.toString());
        return sb.toString();
    }

    //省略get、set方法
}
```

测试类：
```java
public static void main(String[] args) throws CloneNotSupportedException {
        Student student1 = new Student();
        Classroom classroom = new Classroom();
        classroom.setClassName("高三<7>班");
        student1.setName("Jone");
        student1.setAge(20);
        student1.setClassroom(classroom);
        Student student2 = (Student) student1.clone();
        Student student3 = (Student) student1.clone();

        //先观察clone出来的对象地址
        System.out.println("各个对象的地址");
        System.out.println(student1.toString());
        System.out.println(student2.toString());
        System.out.println(student3.toString());


        System.out.println("修改student2和student3的属性");
        student2.setName("Marry");
        student2.setAge(18);
        student3.setName("Stan");
        student3.setAge(19);
        System.out.println(student1);
        System.out.println(student2);
        System.out.println(student3);

        System.out.println("Marry被调到高三<1>班");
        student2.getClassroom().setClassName("高三<1>班");
        System.out.println(student1);
        System.out.println(student2);
        System.out.println(student3);
    }
```
```java
Student类的无参构造器...
各个对象的地址
{"Name":"Jone","Age":20,"Classroom":{"ClassName":"高三<7>班"}Prototype.BlogDemo.Classroom@4554617c}    Prototype.BlogDemo.Student@74a14482
{"Name":"Jone","Age":20,"Classroom":{"ClassName":"高三<7>班"}Prototype.BlogDemo.Classroom@4554617c}    Prototype.BlogDemo.Student@1540e19d
{"Name":"Jone","Age":20,"Classroom":{"ClassName":"高三<7>班"}Prototype.BlogDemo.Classroom@4554617c}    Prototype.BlogDemo.Student@677327b6
修改student2和student3的属性
{"Name":"Jone","Age":20,"Classroom":{"ClassName":"高三<7>班"}Prototype.BlogDemo.Classroom@4554617c}    Prototype.BlogDemo.Student@74a14482
{"Name":"Marry","Age":18,"Classroom":{"ClassName":"高三<7>班"}Prototype.BlogDemo.Classroom@4554617c}    Prototype.BlogDemo.Student@1540e19d
{"Name":"Stan","Age":19,"Classroom":{"ClassName":"高三<7>班"}Prototype.BlogDemo.Classroom@4554617c}    Prototype.BlogDemo.Student@677327b6
Marry被调到高三<1>班
{"Name":"Jone","Age":20,"Classroom":{"ClassName":"高三<1>班"}Prototype.BlogDemo.Classroom@4554617c}    Prototype.BlogDemo.Student@74a14482
{"Name":"Marry","Age":18,"Classroom":{"ClassName":"高三<1>班"}Prototype.BlogDemo.Classroom@4554617c}    Prototype.BlogDemo.Student@1540e19d
{"Name":"Stan","Age":19,"Classroom":{"ClassName":"高三<1>班"}Prototype.BlogDemo.Classroom@4554617c}    Prototype.BlogDemo.Student@677327b6
```

对于输出结果，有下面几个点值得注意:
1.clone对象并不会调用对象的构造方法。
2.clone出来的对象地址都不一样，可以说明当执行完clone方法的时候，会在堆内存中开辟出一块内存给新对象。
3.clone出来的对象属性都一样，包括引用对象的内存地址都一样。所以当修改任意对象的引用变量的时候，会把所有含有被修改的引用变量都更改。

![Show clone method in heap and stack](https://raw.githubusercontent.com/MuggleLee/PicGo/master/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E5%8E%9F%E5%9E%8B%E6%A8%A1%E5%BC%8F/Show%20clone%20method%20in%20heap%20and%20stack.png)

关于第三点，其实就是经典的一个话题：深克隆和浅克隆。
>**浅克隆**：创建一个新对象，新对象的属性和原来对象完全相同，对于非基本类型属性，仍指向原有属性所指向的对象的内存地址。
**深克隆**：创建一个新对象，属性中引用的其他对象也会被克隆，不再指向原有对象地址。

很明显，上面的例子是浅克隆。那深克隆如何做到呢？
首先需要修改Student类和Classroom类
```java
public class Student implements Cloneable {
    private String name;
    private Integer age;
    private Classroom classroom;

    public Student() {
        System.out.println("Student类的无参构造器...");
    }

    @Override
    public String toString() {
        final StringBuilder sb = new StringBuilder("{");
        sb.append("\"Name\":\"")
                .append(name).append('\"');
        sb.append(",\"Age\":")
                .append(age);
        sb.append(",\"Classroom\":")
                .append(classroom);
        sb.append('}').append("    ").append(super.toString());
        return sb.toString();
    }

    //省略get、set方法

    @Override
    protected Object clone() throws CloneNotSupportedException {
        Student student = (Student) super.clone();
        student.setClassroom((Classroom) student.classroom.clone());
        return student;
    }
}

```

```java
public class Classroom implements Cloneable{

    private String className;

    public Classroom() {
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }

    @Override
    public String toString() {
        final StringBuilder sb = new StringBuilder("{");
        sb.append("\"ClassName\":\"")
                .append(className).append('\"');
        sb.append('}').append(super.toString());
        return sb.toString();
    }

    //省略get、set
}

```

输出结果：
```java
Student类的无参构造器...
各个对象的地址
{"Name":"Jone","Age":20,"Classroom":{"ClassName":"高三<7>班"}Prototype.BlogDemo.Classroom@4554617c}    Prototype.BlogDemo.Student@74a14482
{"Name":"Jone","Age":20,"Classroom":{"ClassName":"高三<7>班"}Prototype.BlogDemo.Classroom@1540e19d}    Prototype.BlogDemo.Student@677327b6
{"Name":"Jone","Age":20,"Classroom":{"ClassName":"高三<7>班"}Prototype.BlogDemo.Classroom@14ae5a5}    Prototype.BlogDemo.Student@7f31245a
修改student2和student3的属性
{"Name":"Jone","Age":20,"Classroom":{"ClassName":"高三<7>班"}Prototype.BlogDemo.Classroom@4554617c}    Prototype.BlogDemo.Student@74a14482
{"Name":"Marry","Age":18,"Classroom":{"ClassName":"高三<7>班"}Prototype.BlogDemo.Classroom@1540e19d}    Prototype.BlogDemo.Student@677327b6
{"Name":"Stan","Age":19,"Classroom":{"ClassName":"高三<7>班"}Prototype.BlogDemo.Classroom@14ae5a5}    Prototype.BlogDemo.Student@7f31245a
Marry被调到高三<1>班
{"Name":"Jone","Age":20,"Classroom":{"ClassName":"高三<7>班"}Prototype.BlogDemo.Classroom@4554617c}    Prototype.BlogDemo.Student@74a14482
{"Name":"Marry","Age":18,"Classroom":{"ClassName":"高三<1>班"}Prototype.BlogDemo.Classroom@1540e19d}    Prototype.BlogDemo.Student@677327b6
{"Name":"Stan","Age":19,"Classroom":{"ClassName":"高三<7>班"}Prototype.BlogDemo.Classroom@14ae5a5}    Prototype.BlogDemo.Student@7f31245a
```

由结果可以看出，使用强克隆可以为引用对象开辟一块内存地址而不是像浅克隆那样指向同一个引用对象。

# 总结：
>设计原型模式需要注意浅克隆和深克隆。
