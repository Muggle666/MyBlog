

## What:




## Why:
##### 优点：


##### 缺点：


## Where:


## How:
职责链模式主要包含以下角色：

**抽象处理者（Handler）角色：** 定义一个处理请求的接口，包含抽象处理方法和一个后继连接。

**具体处理者（Concrete Handler）角色：** 实现抽象处理者的处理方法，判断能否处理本次请求，如果可以处理请求则处理，否则将该请求转给它的后继者。

**客户类（Client）角色：** 创建处理链，并向链头的具体处理者对象提交请求，它不关心处理细节和请求的传递过程。


ReviewPerson（抽象处理者角色）：
```java
public abstract class ReviewPerson {
    protected ReviewPerson person;

    abstract void handle(String program);

    public ReviewPerson getPerson() {
        return person;
    }

    public void setPerson(ReviewPerson person) {
        this.person = person;
    }
}
```
Tester、CTO、Boss（具体处理者角色）：
```java
public class Tester extends ReviewPerson{
    @Override
    void handle(String program) {
        if("没有Bug的功能！".equals(program)){
            System.out.println("测试人员测试后没问题，提交给技术总监...");
            getPerson().handle(program);
        }else {
            System.out.println("有Bug呀，再改改！");
        }
    }
}
public class CTO extends ReviewPerson{
    @Override
    void handle(String program) {
        if("没有Bug的功能！".equals(program)){
            System.out.println("技术总监测试后没问题，提交给老板...");
            getPerson().handle(program);
        }else {
            System.out.println("有Bug呀，再改改！");
        }
    }
}
public class Boss extends ReviewPerson{
    @Override
    void handle(String program) {
        if("没有Bug的功能！".equals(program)){
            System.out.println("功能完成，可以上线了！");
        }else {
            System.out.println("有Bug呀，再改改！");
        }
    }
}
```
Programmer（客户类角色）：
```java
public class Programmer {
    public static void main(String[] args) {
        ReviewPerson tester = new Tester();
        ReviewPerson cto = new CTO();
        ReviewPerson boss = new Boss();

        tester.setPerson(cto);
        cto.setPerson(boss);

        tester.handle("没有Bug的功能！");
    }
}
```
输出结果：
```java
测试人员测试后没问题，提交给技术总监...
技术总监测试后没问题，提交给老板...
功能完成，可以上线了！

```


# 总结

参考资料:
