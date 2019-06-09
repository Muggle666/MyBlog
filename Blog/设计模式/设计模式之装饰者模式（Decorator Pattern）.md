# 设计模式之装饰者模式
## What:

装饰者模式又名包装(Wrapper)模式。装饰者模式动态地将责任附加到对象身上。若要扩展功能，装饰者提供了比继承更有弹性的替代方案。

## Why:
**优点：**
1.装饰者模式比继承灵活性，在不改变原有对象的情况下给对象扩展功能，符合开闭原则。
>继承关系是静态的，在编译的时候就已经决定了行为，不便于控制增加行为的方式和时机。

2.装饰者模式可以动态使用不同的装饰类排列组合，创造出多样的行为组合。

**缺点：**
>1.装饰模式会导致设计出大量的ConcreteDecorator类，增加系统的复杂性。
>2.对于多次装饰的对象，一旦出现错误，排错繁琐；

## Where:

>1.在不影响其他对象的情况下，以动态、透明的方式给单个对象添加职责。
>2.需要动态地给一个对象增加功能，这些功能也可以动态地被撤销。  
>3.当不能采用继承的方式对系统进行扩充或者采用继承不利于系统扩展和维护时。

## How:

在学习使用装饰者模式之前，先了解几个重要角色。

**Component（抽象构件）**：定义需要实现业务的抽象方法。

**ConcreteComponent（具体构件）**：实现Component接口，用于定义具体的构建对象，可以给它增加额外的职责（方法）。

**Decorator（抽象装饰类）**：实现Component接口，并创建Component实例对象。用于给具体构件(ConcreteComponent)增加职责。

**ConcreteDecorator（具体装饰类）**：抽象装饰类的子类，负责向构件添加新的职责。每一个具体装饰类都定义了一些新的行为，它可以调用在抽象装饰类中定义的方法，并可以增加新的方法用以扩充对象的行为。

![装饰者模式UML](https://raw.githubusercontent.com/MuggleLee/PicGo/master/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E8%A3%85%E9%A5%B0%E8%80%85%E6%A8%A1%E5%BC%8F/Pattern-Decorator.png)


示例：以点餐为例，我在麦*劳点餐可以根据我需求增加食物种类和数量。


抽象构件角色：
```java
public interface Food {
    String getDescription();
}
```
具体构件角色：
```java
public class BasicSet implements Food{
    @Override
    public String getDescription() {
        return "汉堡 + 可乐";
    }
}
```
抽象装饰类角色：
```java
public abstract class Decorator implements Food {

    private Food food;

    public Decorator(Food food) {
        this.food = food;
    }

    @Override
    public String getDescription() {
        return this.food.getDescription();
    }

}
```

具体装饰类角色：
```java
public class FrenchFries extends Decorator {
    public FrenchFries(Food food) {
        super(food);
    }
    @Override
    public String getDescription() {
        return super.getDescription() + " + 薯条";
    }
}
```
```java
public class FriedChicken extends Decorator {
    public FriedChicken(Food food) {
        super(food);
    }
    @Override
    public String getDescription() {
        return super.getDescription() + " + 炸鸡";
    }
}
```
```java
public class IceCream extends Decorator {
    public IceCream(Food food) {
        super(food);
    }
    @Override
    public String getDescription() {
        return super.getDescription() + " + 冰淇淋";
    }
}

```

测试类：
```java
1. public class Test {
2.     public static void main(String[] args) {
3.         Food food = new BasicSet();
4.         Decorator setMealA = new FrenchFries(food);
5.         setMealA = new FrenchFries(setMealA);
6.         setMealA = new FriedChicken(setMealA);
7.         setMealA = new IceCream(setMealA);
8.         System.out.println("套餐A：" + setMealA.getDescription());
9.     }
10. }
```
输出结果：
```java
套餐A：汉堡 + 可乐 + 薯条 + 薯条 + 炸鸡 + 冰淇淋
```
![https://raw.githubusercontent.com/MuggleLee/PicGo/master/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E8%A3%85%E9%A5%B0%E8%80%85%E6%A8%A1%E5%BC%8F/Food.png](https://raw.githubusercontent.com/MuggleLee/PicGo/master/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E8%A3%85%E9%A5%B0%E8%80%85%E6%A8%A1%E5%BC%8F/Food.png)


第3行创建基础的构件（点一份套餐，有汉堡和可乐），第4~7行分别加上需要装饰的类（还想吃两份薯条，一份炸鸡和一份冰淇淋）。由此可以看出，装饰者模式很方便的根据需求动态的改变装饰部分。


不过各位有没有觉得，装饰者模式和建造者模式很相似呢？相似点都是内部把需求拼装好之后才展示出来，那什么情况下应该使用装饰者模式，什么情况下使用构建者模式呢？


|装饰者模式|建造者模式|
|-|-|
|装饰物与被装饰物继承自同一组件类，组件对象均可由装饰物在外部按顺序装饰；|被建造者集成一个成员组成稳定的接口，并对成员内容做具体实现。|
|各装饰物记录自己装饰了谁，其功能函数除了实现自己的功能外，还需要调用被自己装饰的组件的功能函数|被创建对象的成员有一个指挥类稳定的按顺序建造，指挥类隔离建造过程及客户代码；|
|针对构建过程不稳定的情况|针对建造过程十分稳定的情况|



参考资料：

[https://blog.csdn.net/wwwdc1012/article/details/82764333](https://blog.csdn.net/wwwdc1012/article/details/82764333)

[https://www.jianshu.com/p/d7f20ae63186](https://www.jianshu.com/p/d7f20ae63186)