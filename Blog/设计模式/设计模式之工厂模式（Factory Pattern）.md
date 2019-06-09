# 设计模式之工厂模式

## What:

> 工厂模式又称多态性工厂模式。定义一个创建对象的接口，让其子类自己决定实例化哪一个工厂类，工厂模式使其创建过程延迟到子类进行。

## Where:
> 1.创建对象需要大量重复代码；
2.客户端不关心实例如何被创建、实现的细节；
3.一个类通过子类来指定创建哪个对象。

## Why:
#### 优点：
> 1.一个调用者想创建一个对象，只要知道其名称就可以了。
2.扩展性高，如果想增加一个产品，只要扩展一个工厂类就可以，而不需要改动先有的类，符合“开闭原则”。
3.屏蔽产品的具体实现，调用者只关心产品的接口。
4.每个具体工厂类只负责创建对应的产品，符合单一职责原则。

#### 缺点：
>每次增加一个产品时，都需要增加一个具体类和对象实现工厂，使得系统中类的个数成倍增加，在一定程度上增加了系统的复杂度，同时也增加了系统具体类的依赖。

## How:
**抽象工厂(Abstract Factory)角色：** 是工厂方法模式的核心，与应用程序无关。任何在模式中创建的对象的工厂类必须实现这个接口。
**具体工厂(Concrete Factory)角色 ：** 这是实现抽象工厂接口的具体工厂类，包含与应用程序密切相关的逻辑，并且受到应用程序调用以创建某一种产品对象。
**抽象产品(Abstract Product)角色 ：** 工厂方法模式所创建的对象的超类型，也就是产品对象的共同父类或共同拥有的接口。
**具体产品(Concrete Product)角色 ：** 这个角色实现了抽象产品角色所定义的接口。某具体产品有专门的具体工厂创建，它们之间往往一一对应。

![Pattern-Factory](https://raw.githubusercontent.com/MuggleLee/PicGo/master/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E5%B7%A5%E5%8E%82%E6%A8%A1%E5%BC%8F/Pattern-Factory.png)

### 示例：创建汽车工厂，根据需求获取不同品牌汽车。

#### CarFactory抽象类（抽象工厂）：
```java
public abstract class CarFactory {
    abstract Car getCar();
}
```
#### BenzFactory类，AudiFactory类，BmwFactory类（具体工厂）：
```java
public class BenzFactory extends CarFactory{
    @Override
    Car getCar() {
        return new Benz();
    }
}

public class AudiFactory extends CarFactory {
    @Override
    Car getCar() {
        return new Audi();
    }
}

public class BmwFactory extends CarFactory {
    @Override
    Car getCar() {
        return new Bmw();
    }
}
```
#### Car类（抽象产品）：
```java
public abstract class Car {
    public abstract void produce();
}
```
#### Audi类，Bmw类，Benz类（具体产品）：
```java
public class Audi extends Car{
    @Override
    public void produce() {
        System.out.println("制造奥迪...");
    }
}

public class Bmw extends Car {
    @Override
    public void produce() {
        System.out.println("制造宝马...");
    }
}

public class Benz extends Car {
    @Override
    public void produce() {
        System.out.println("制造奔驰...");
    }
}
```

Test：测试类
```java
public class Test {
    public static void main(String[] args) {
        //制造奔驰
        CarFactory benzFactory = new BenzFactory();
        benzFactory.getCar().produce();

        //制造宝马
        CarFactory bmwFactory = new BmwFactory();
        bmwFactory.getCar().produce();

        //制造奥迪
        CarFactory audiFactory = new AudiFactory();
        audiFactory.getCar().produce();
    }
}
```
输出结果：
```java
制造奔驰...
制造宝马...
制造奥迪...
```

示例代码的UML图：

![Sample](https://raw.githubusercontent.com/MuggleLee/PicGo/master/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E5%B7%A5%E5%8E%82%E6%A8%A1%E5%BC%8F/Pattern-Factory-sample.png)

## 工厂模式的特例：简单工厂模式

>GOF在《设计模式》一书中将工厂模式分为两类：工厂方法模式与抽象工厂模式。而简单工厂模式看为工厂方法模式的一种特例，两者归为一类。 

与工厂模式不一样的是，简单工厂模式只有**抽象产品类，具体产品类和一个工厂类**。
![简单工厂模式UML](https://raw.githubusercontent.com/MuggleLee/PicGo/master/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E5%B7%A5%E5%8E%82%E6%A8%A1%E5%BC%8F/%E7%AE%80%E5%8D%95%E5%B7%A5%E5%8E%82%E6%A8%A1%E5%BC%8F/Pattern-SimpleFactory.png)

示例：创建蛋糕工厂，根据客户端传的参数实例化相应的对象。

Cake类：

```java
public abstract class Cake {
    abstract void produce();
}
```
DurianCake、StrawberryCake、ChocolateCake类：
```java
public class DurianCake extends Cake {
    @Override
    void produce() {
        System.out.println("制作榴莲蛋糕...");
    }
}
public class StrawberryCake extends Cake {
    @Override
    void produce() {
        System.out.println("制作草莓蛋糕...");
    }
}
public class ChocolateCake extends Cake {
    @Override
    void produce() {
        System.out.println("制作巧克力蛋糕...");
    }
}
```
CakeFactory类：
```java
public class CakeFactory {
    public Cake getCake(String cakeName) throws Exception {
          //第一种方式：通过if...else...语句判断
        if ("榴莲蛋糕".equals(cakeName)) {
            return new DurianCake();
        } else if ("草莓蛋糕".equals(cakeName)) {
            return new StrawberryCake();
        } else if ("巧克力蛋糕".equals(cakeName)) {
            return new ChocolateCake();
        } else {
            throw new Exception("输入有误，请重新输入！");
        }

        //第二种方式：通过switch...case...判断
//        switch (cakeName) {
//            case "榴莲蛋糕":
//                return new DurianCake();
//            case "草莓蛋糕":
//                return new StrawberryCake();
//            case "巧克力蛋糕":
//                return new ChocolateCake();
//            default:
//                throw new Exception("输入有误，请重新输入！");
//        }
    }
    
    //第三种方式：通过反射创建对象
//    public Cake getCake(Class obj){
//        Cake cake = null;
//        try {
//            cake = (Cake) Class.forName(obj.getName()).newInstance();
//        } catch (InstantiationException e) {
//            e.printStackTrace();
//        } catch (IllegalAccessException e) {
//            e.printStackTrace();
//        } catch (ClassNotFoundException e) {
//            e.printStackTrace();
//        }
//        return cake;
//    }
}
```
Test类：
```java
public class Test {
    public static void main(String[] args) throws Exception {
        //第一种、第二种方式的测试代码
        String cakeName = "榴莲蛋糕";
        CakeFactory cake = new CakeFactory();
        cake.getCake(cakeName).produce();
        //第三种的测试代码
//        CakeFactory cakeFactory = new CakeFactory();
//        cakeFactory.getCake(DurianCake.class).produce();
    }
}
```
输出结果：
```java
制作榴莲蛋糕...
```
![SimpleFactoryPatternSample](https://raw.githubusercontent.com/MuggleLee/PicGo/master/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E5%B7%A5%E5%8E%82%E6%A8%A1%E5%BC%8F/%E7%AE%80%E5%8D%95%E5%B7%A5%E5%8E%82%E6%A8%A1%E5%BC%8F/Pattern-SimpleFactory-Sample.png)

## 总结

工厂模式虽然遵守开闭原则，又保持了封装对象创建过程的优点，但工厂模式的缺点也是显而易见的，就是每增加一个产品类，就需要增加一个对应的工厂类，增加了额外的开发量。相对于工厂模式，简单工厂模式的优点是不用每个产品类都对应一个产品工厂类，简化开发流程，但是缺点是如果新增产品类就需要修改工厂类，违背开闭原则，不过可以通过反射机制解决这个问题。