# 设计模式之模版模式
## What:
>定义一个操作中的算法骨架，而将算法的一些步骤延迟到子类中，使得子类可以不改变该算法结构的情况下重定义该算法的某些特定步骤。



## Why:
#### 优点：
1.封装不变部分，扩展可变部分。 
2.提取公共代码，便于维护。 
3.行为由父类控制，子类实现，符合开闭原则。
4.代码复用，减少代码冗余。

#### 缺点：
1.每一个不同的实现都需要增加子类，导致类数目增加，增加类系统的复杂度；
2.由于继承的关系，父类增加或者修改抽象方法，子类都需要修改。

## Where:
1.多个类共有的方法，且逻辑相同。
2.一次性实现一个算法的不变部分，并将可变的行为留给子类实现。

## How:

模版模式的角色比较简单，只有抽象父类和具体子类。

**AbstractClass（抽象父类）：** 定义模版方法，也就是定义一些公共的行为，将可变的行为抽象，让子类具体实现。

**ConcreteClass（具体子类）：** 实现抽象父类，具体实现可变的行为。

![模版模式UML](https://raw.githubusercontent.com/MuggleLee/PicGo/master/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E6%A8%A1%E7%89%88%E6%A8%A1%E5%BC%8F/TemplatePattern.png)

在写具体代码之前，先了解一下**钩子方法**。

#### 钩子方法是什么呢？
>就是在抽象类中定义一个方法，默认不做任何事，子类可以根据实际情况要不要覆盖它，从而改变行为，该方法称为“钩子”。

示例：模拟制作咖啡店家接收订单的过程。父类Coffee定义了共有的方法，如加热水，研磨咖啡豆，同时也定义了可变部分，有的消费者不喜欢加奶加糖，所以定义了钩子方法isAddMilkFlag和isAddSugarFlag判断是否需要加奶和糖。而子类继承抽象父类根据实际需要判断是否需要添加奶和糖。

Coffee（抽象父类）：
```java
public abstract class Coffee {

    boolean addSugarFlag = false;

    boolean addMilkFlag = false;

    public boolean isAddMilkFlag() {
        return addMilkFlag;
    }

    public boolean isAddSugarFlag() {
        return addSugarFlag;
    }

    Coffee prepareHotWater(){
        System.out.println("准备热水");
        return this;
    }

    Coffee grindCoffeeBean(){
        System.out.println("研磨咖啡豆");
        return this;
    }

    void addSugar(){
        System.out.println("加糖");
    }

    void addMilk(){
        System.out.println("加奶");
    }

    Coffee make(String coffeeName){
        Coffee coffee = prepareHotWater().grindCoffeeBean();
        if(isAddMilkFlag()){
            coffee.addMilk();
        }
        if(isAddSugarFlag()){
            coffee.addSugar();
        }
        System.out.println("制作完成！这是一杯"
                + (isAddSugarFlag() ? "加" : "不加") + "糖,"
                + (isAddMilkFlag() ? "加" : "不加") + "奶"
                + "的" + coffeeName);
        return coffee;
    }
}
```
Cappuccino类和Latte类:
```java
public class Cappuccino extends Coffee {

    String coffeeName = "卡布奇诺";

    Coffee make(){
        return super.make(this.coffeeName);
    }

    @Override
    public boolean isAddSugarFlag() {
        return true;
    }
}

public class Latte extends Coffee {
    String coffeeName = "拿铁";

    Coffee make(){
        return super.make(this.coffeeName);
    }

    @Override
    public boolean isAddMilkFlag() {
        return true;
    }

    @Override
    public boolean isAddSugarFlag() {
        return true;
    }
}
```
Test:测试类
```java
public class Test {
    public static void main(String[] args) {
        System.out.println("******  下订单：一杯加糖,不加奶的热卡布奇诺  ******");
        Cappuccino cappuccino = new Cappuccino();
        cappuccino.make();
        System.out.println("******  下订单：一杯加糖,加奶的热拿铁  ******");
        Latte latte = new Latte();
        latte.make();
    }
}
```
输出结果：
```java
******  下订单：一杯加糖,不加奶的热卡布奇诺  ******
准备热水
研磨咖啡豆
加糖
制作完成！这是一杯加糖,不加奶的卡布奇诺
******  下订单：一杯加糖,加奶的热拿铁  ******
准备热水
研磨咖啡豆
加奶
加糖
制作完成！这是一杯加糖,加奶的拿铁
```



# 总结
使用模版模式可以很方便的将共有的代码提取出来，能降低代码的冗余，加快开发效率。在实际开发中也经常用到模版模式的设计思想。譬如前端html页面，页头和页尾总是一样的，那么可以创建页头(header.html)和页尾(footer.html)模版，在每个html页面都直接引用，这样的话，既能够节省时间，又能大大的减少代码量。


参考资料:
[https://www.runoob.com/design-pattern/template-pattern.html](https://www.runoob.com/design-pattern/template-pattern.html)