# 设计模式之观察者模式
## What:

>观察者模式又叫做发布-订阅模式，定义了对象间一对多的依赖关系，使得当对象状态发生变化时，所有依赖它的对象都会收到通知并且自动更新自己。

![ObserverPattern-Sample](https://raw.githubusercontent.com/MuggleLee/PicGo/master/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E8%A7%82%E5%AF%9F%E8%80%85%E6%A8%A1%E5%BC%8F/Pattern-Observer.png)

## Why:
优点：
观察者和被观察者是抽象耦合的
实现广播机制，被观察者可以群发消息给观察者。


缺点：
如果一个被观察者对象有很多的直接和间接的观察者的话，将所有的观察者都通知到会花费很多时间，影响执行效率。
如果被观察者和观察者之间有依赖关系，可能会出现循环引用。



## Where:

一个对象必须通知其他对象，而并不知道这些对象是谁。
对象间存在一对多关系，一个对象的状态发生改变会影响其他对象。

## How:

首先了解与观察者模式相关的4个概念：

**Subject（抽象目标）：** 接口或抽象类。定义了动态增加、删除以及通知观察者对象的方法，职责就是管理和通知观察者。持有观察者对象的集合。

**ConcreteSubject（具体目标）：** 具体目标类，它实现抽象目标中的通知方法，当具体主题的内部状态发生改变时，通知所有注册过的观察者对象。

**Observer（抽象观察者）：** 提供一个接口，定义了观察者收到通知时更新自己的方法。

**ConcreteObserver（具体观察者）：** 实现抽象观察者中定义的抽象方法，以便在得到目标的更改通知时更新自身的状态。

![ObserverPattern-UML](https://raw.githubusercontent.com/MuggleLee/PicGo/master/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E8%A7%82%E5%AF%9F%E8%80%85%E6%A8%A1%E5%BC%8F/Pattern-Observer-UML.png)

示例：当我们在电商平台关注的商品或者加入购物车的商品降价的时候，平台会发信息给我们，这就是典型的订阅-发布模式，即观察者模式。

### 通过自定义抽象目标和抽象观察者

Observerable（抽象目标）：
```java
public interface Observerable {
    List<Observer> list = new ArrayList<>();

    void notifyObserver(String product, BigDecimal money);

    void registerObserver(Observer observer);

    void removeObserver(Observer observer);
}
```
Mall（具体目标）:
```java
public class Mall implements Observerable {

    private final String mallName = "京西";

    //通知所有观察者
    @Override
    public void notifyObserver(String product, BigDecimal money) {
        for (int i = 0; i < list.size(); i++) {
            Observer observer = list.get(i);
            observer.update(mallName,product,money);
        }
    }
    
    //注册保存观察者
    @Override
    public void registerObserver(Observer observer) {
        list.add(observer);
    }

    //移除观察者
    @Override
    public void removeObserver(Observer observer) {
        list.remove(observer);
    }
}
```
Observer（抽象观察者）:
```java
public interface Observer {
    void update(String mallName,String product, BigDecimal money);
}
```
User（观察者）:
```java
public class User implements Observer {

    private String userName;

    public User(String name) {
        this.userName = name;
    }

    //更新消息
    @Override
    public void update(String mallName, String product, BigDecimal money) {
        System.out.println(userName + "接收到一条来自" + mallName + "的消息：您关注的商品【" + product + "】降价了！比您加入购物车的时候降了" + money + "元！");
    }
}
```
Test：测试类
```java
public class Test {
    public static void main(String[] args) {
        Observerable mall = new Mall();
        //添加3个订阅者
        Observer user1 = new User("MuggleLee");
        Observer user2 = new User("Josn");
        Observer user3 = new User("Pakho");
        //注册保存订阅者
        mall.registerObserver(user1);
        mall.registerObserver(user2);
        mall.registerObserver(user3);
        //通知所有的观察者
        mall.notifyObserver("iPhone XR",new BigDecimal(100));

        System.out.println();
        //移除一位订阅者
        mall.removeObserver(user3);
        //通知所有的观察者
        mall.notifyObserver("iPhone XS",new BigDecimal(300));
    }
}
```
输出结果：
```java
MuggleLee接收到一条来自京西的消息：您关注的商品【iPhone XR】降价了！比您加入购物车的时候降了100元！
Josn接收到一条来自京西的消息：您关注的商品【iPhone XR】降价了！比您加入购物车的时候降了100元！
Pakho接收到一条来自京西的消息：您关注的商品【iPhone XR】降价了！比您加入购物车的时候降了100元！

MuggleLee接收到一条来自京西的消息：您关注的商品【iPhone XS】降价了！比您加入购物车的时候降了300元！
Josn接收到一条来自京西的消息：您关注的商品【iPhone XS】降价了！比您加入购物车的时候降了300元！
```

### 使用JDK提供的Observable抽象类和Observer接口
Mall（被观察者）：
```java
public class Mall extends Observable {

    public final String mallName = "京西";

    public void send(Message message){
        setChanged();//标记被观察者要发生消息
        notifyObservers(message);//通知所有观察者
    }
}
```
Message（消息类）：消息内容的对象
```java
public class Message {
    private String productName;
    private BigDecimal money;

    public Message() {
    }

    public Message(String productName, BigDecimal money) {
        this.productName = productName;
        this.money = money;
    }

    public String getProductName() {
        return productName;
    }

    public BigDecimal getMoney() {
        return money;
    }
}
```
User（观察者）：
```java
public class User implements Observer {

    private String userName;

    public User(String name) {
        this.userName = name;
    }

    @Override
    public void update(Observable o, Object arg) {
        Mall mall = (Mall) o;
        Message message = (Message) arg;
        System.out.println(userName + "接收到一条来自" + mall.mallName + "的消息：您关注的商品‘"
                + message.getProductName() + "'降价了！比您加入购物车的时候降了" + message.getMoney() + "元！");
    }
}
```
Test：测试类
```java
public class Test {
    public static void main(String[] args) {
        Observable mall = new Mall();

        Observer user1 = new User("MuggleLee");
        Observer user2 = new User("Josn");
        Observer user3 = new User("Pakho");

        mall.addObserver(user1);
        mall.addObserver(user2);
        mall.addObserver(user3);

        Message message = new Message("iPhone XR",new BigDecimal(100));

        ((Mall) mall).send(message);

        System.out.println();

        mall.deleteObserver(user3);
        Message message2 = new Message("iPhone XS",new BigDecimal(300));
        ((Mall) mall).send(message2);
    }
}
```
输出结果：
```jav
Pakho接收到一条来自京西的消息：您关注的商品‘iPhone XR'降价了！比您加入购物车的时候降了100元！
Josn接收到一条来自京西的消息：您关注的商品‘iPhone XR'降价了！比您加入购物车的时候降了100元！
MuggleLee接收到一条来自京西的消息：您关注的商品‘iPhone XR'降价了！比您加入购物车的时候降了100元！

Josn接收到一条来自京西的消息：您关注的商品‘iPhone XS'降价了！比您加入购物车的时候降了300元！
MuggleLee接收到一条来自京西的消息：您关注的商品‘iPhone XS'降价了！比您加入购物车的时候降了300元！
```

# 总结
根据项目实际需求选择自定义还是使用JDK提供的Observer接口和Observable抽象类吧。
