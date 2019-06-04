# 设计模式之享元模式
## What:

>使用共享的方式高效的支持大量的细粒度的对象。

## Why:
优点：
1.减少创建的对象数量，减少内存占用和提高性能；
2.享元模式的外部状态相对独立，而且不会影响其内部状态，从而使得享元对象可以在不同的环境中被共享。

缺点：
1.享元模式使得系统更加复杂，需要分离出内部状态和外部状态，这使得程序的逻辑复杂化； 
2.为了使对象可以共享，享元模式需要将享元对象的状态外部化，而读取外部状态使得运行时间变长。
## Where:

1、系统有大量相似对象。 
2、需要缓冲池的场景。

## How:

在学习使用享元模式之前，我们需要先了解几个概念。

**FlyWeight：定义共享对象的功能。**

**ConcreteFlyWeight：共享对象的具体实现。**

**FlyWeightFactory：封装共享对象工厂。**


![](https://raw.githubusercontent.com/MuggleLee/PicGo/master/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E4%BA%AB%E5%85%83%E6%A8%A1%E5%BC%8F/Pattern-Flyweight.png)

示例：以王者荣耀的角色创建为例，通过创建角色的过程进一步体会享元模式。(FlyWeight接口并非一定需要，分了简便Demo中省略)

#### Hero类：英雄的属性
```java
public class Hero {

    private String heroClass;//英雄职业

    private String name;//角色名字

    private long attack;//攻击力

    private long defence;//防御力

    public Hero() {
    }

    //省略get、set方法
    
    public void create() {
        System.out.println("职业：" + this.getHeroClass() + "\n"
                + "创建的角色：" + this.getName() + "\n"
                + "角色属性：" + "\n"
                + "攻击力：" + this.getAttack() + "\n"
                + "防御力：" + this.getDefence() + "\n");
    }
}
```
#### HeroFactory类：创建英雄的工厂
```java
public class HeroFactory {

    public HashMap<String,Hero> map = new HashMap();

    public Hero factory(String heroClass){
        Hero hero = map.get(heroClass);
        if(hero == null){
            hero = new Hero();
            map.put(heroClass,hero);
        }
        return hero;
    }
}
```
#### Test类：测试类
```java
public class Test {
    public static void main(String[] args) {
        HeroFactory heroFactory = new HeroFactory();

        String fashi = "法师";
        String sheshou = "射手";

        Hero hero1 = heroFactory.factory(fashi);
        hero1.setHeroClass(fashi);
        hero1.setName("王昭君");
        hero1.setAttack(1000);
        hero1.setDefence(100);
        hero1.create();


        Hero hero2 = heroFactory.factory(fashi);
        hero2.setHeroClass(fashi);
        hero2.setName("嬴政");
        hero2.setAttack(1200);
        hero2.setDefence(180);
        hero2.create();


        Hero hero3 = heroFactory.factory(sheshou);
        hero3.setHeroClass(sheshou);
        hero3.setName("后裔");
        hero3.setAttack(1500);
        hero3.setDefence(100);
        hero3.create();

        System.out.println("输出各对象的地址：");
        System.out.println(hero1);
        System.out.println(hero2);
        System.out.println(hero3);
    }
}
```
输出结果：
```java
职业：法师
创建的角色：王昭君
角色属性：
攻击力：1000
防御力：100

职业：法师
创建的角色：嬴政
角色属性：
攻击力：1200
防御力：180

职业：射手
创建的角色：后裔
角色属性：
攻击力：1500
防御力：100

输出各对象的地址：
FlyweightPattern.Hero@4554617c
FlyweightPattern.Hero@4554617c
FlyweightPattern.Hero@74a14482
```

# 总结

通过输出的对象地址可知，同一职业的英雄对象不会重复创建。也就是说，在享元模式下，可以避免重复创建相同或者相似的对象，这样可以减少内存和提高程序的效率。



参考资料：

《大话设计模式》

[https://www.runoob.com/design-pattern/flyweight-pattern.html](https://www.runoob.com/design-pattern/flyweight-pattern.html)

[http://c.biancheng.net/view/1371.html](http://c.biancheng.net/view/1371.html)





