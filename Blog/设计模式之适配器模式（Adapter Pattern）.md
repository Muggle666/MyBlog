# 设计模式之适配器模式
## What:

>将一个类的接口转换成客户希望的另外一个接口，使得原本由于接口不兼容而不能一起工作的那些类能一起工作。


## Why:
优点：
1.客户端通过适配器可以透明地调用目标接口；
2.复用了现存的类，不需要修改原有代码而重用现有的适配者类，符合开闭原则；
3.将目标类和适配者类解耦，解决了目标类和适配者类接口不一致的问题；

缺点：
1.Java不支持多重继承，一次最多可以继承一个适配器类；
2.过多的使用适配器，会让系统非常零乱，不易整体进行把握；
3.类适配器模式中的目标抽象类只能为接口，不能为类，其使用有一定的局限性；
4.类的适配器模式使用对象继承的方式，高耦合，灵活性低；
5.对象的适配器模式需要引入对象实例，使用复杂；


## Where:
1.系统需要使用现有的类，而这些类的接口不符合系统的接口；
2.多个组件功能类似，但接口不统一且可能会经常切换时，可使用适配器模式，使得客户端可以以统一的接口使用它们；

## How:

适配器模式包含以下几个重要角色：

**Target（目标接口）**：当前系统业务所期待的接口，它可以是抽象类或接口。

**Adaptee（适配者类）**：已存在的将被适配的类。

**Adapter（适配器类）**：适配器类是适配器模式的核心。通过继承或引用适配者的对象，把适配者接口转换成目标接口，让客户按目标接口的格式访问适配者。

值得注意的是，适配器模式有**类的适配器模式**和**对象的适配器模式**两种不同的形式。

通过以下UML有助于更好理解类的适配器模式和对象的适配器模式：

![两种适配器模式的UML](https://raw.githubusercontent.com/MuggleLee/PicGo/master/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E9%80%82%E9%85%8D%E5%99%A8%E6%A8%A1%E5%BC%8F/Pattern-Adapter.png)


### 类的适配器模式

Target类:
```java
public interface Target {
    void targetMethod();
}
```
Adaptee类：
```java
public class Adaptee {
    public void adapteeMethod(){
        System.out.println("被适配类的方法");
    }
}
```
Adapter类：
```java
public class Adapter extends Adaptee implements Target {
    @Override
    public void targetMethod() {
        super.adapteeMethod();
    }
}
```
ClassAdapterTest测试类：
```java
public class ClassAdapterTest {
    public static void main(String[] args) {
        Adapter adapter = new Adapter();
        adapter.targetMethod();
    }
}
```
输出结果：
```java
被适配类的方法
```


### 对象的适配器模式
Target类:
```java
public interface Target {
    void targetMethod();
}
```
Adaptee类：
```java
public class Adaptee {
    public void adapteeMethod(){
        System.out.println("被适配类的方法");
    }
}
```
Adapter类：
```java
public class Adapter implements Target{

    private Adaptee adaptee = new Adaptee();

    @Override
    public void targetMethod() {
        adaptee.adapteeMethod();
    }
}
```
ObjectAdapterTest测试类：
```java
public class ObjectAdapterTest {
    public static void main(String[] args) {
        Adapter adapter = new Adapter();
        adapter.targetMethod();
    }
}
```
输出结果：
```java
被适配类的方法
```

让我举个栗子：在项目中有支付模块，但是支付的接口各个银行都不一样，那怎么才能在项目中调用多个银行的支付接口呢？可以通过适配器类中的方法调用各个银行的对象。

假设银行只提供一个对象，通过实例化就可以调用方法执行支付操作，执行银行的接口会返回状态码。
示例中分别用BankA和BankB代表两个不同的银行。

支付的接口（IPay）：
```java
public interface IPay {
    String pay(String bankName) throws Exception;
}
```
支付的适配器（PayAdapter）：
```java
public class PayAdapter implements IPay {

    private BankA bankA = new BankA();//适配者类
    private BankB bankB = new BankB();//适配者类

    @Override
    public String pay(String bankName) throws Exception {
        if ("bankA".equals(bankName) && "200".equals(bankA.DeductMoney())) {
            System.out.println("银行A：扣款成功！");
            return "200";
        } else if ("bankB".equals(bankName) && "200".equals(bankB.DeductMoney())) {
            System.out.println("银行B：扣款成功！");
            return "200";
        } else {
            throw new Exception("支付失败！请重试！");
        }
    }
}
```
测试方法（PayTest）：
```java
public class PayTest {

    public final static String bankName = "bankA";

    public static void main(String[] args) throws Exception {
        IPay iPay = new PayAdapter();
        iPay.pay(bankName);
    }
}
```
输出结果：
```java
银行A：扣款成功！
```

## 总结

类的适配器模式和对象的适配器模式的区别

||类的适配器模式|对象的适配器模式|
|-|-|-|
|定义|把适配者类的API转换成为目标类的API|不是使用继承关系连接到Adaptee类，而是使用委派关系连接到Adaptee类。在Adapter类包装Adaptee实例，然后通过实例调用适配者类。|
|实现方式|实现接口，继承对象|实现接口，实例化对象|
|优点|1.仅仅引入了一个对象，并不需要额外的引用来间接得到Adaptee。|1.一个对象适配器可以把多个不同的适配者适配到同一个目标；</br>2.可以适配一个适配者的子类，由于适配器和适配者之间是关联关系，根据“里氏代换原则”，适配者的子类也可通过该适配器进行适配。|
|缺点|1.Java只允许继承一个类，因此至多只能适配一个适配者类；</br>2.目标抽象类只能为接口，不能为类，其使用有一定的局限性;|1.需要额外的引用来间接得到Adaptee|


参考资料：
《大话设计模式》
[https://blog.csdn.net/wwwdc1012/article/details/82780560](https://blog.csdn.net/wwwdc1012/article/details/82780560)