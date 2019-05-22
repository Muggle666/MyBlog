# 设计模式之建造者模式

## What:
建造者模式是将一个复杂的**对象的构建与表示**分离，使得同样的构建过程可以创建不同的表示。建造者模式隐藏了复杂对象的创建过程，它把复杂对象的创建过程加以抽象，通过子类继承或者重载的方式，动态的创建具有复合属性的对象。

## Why:
优点：
1.遵循开闭原则。
2.对象的建造和表示分离，实现了解耦。
3.隐藏了对象的建造细节，用户只需关心产品的表示，而不需要了解是如何创建产品的。
缺点：
1.如果构造者多，会有很多的建造类，难以维护。
2.产品的组成部分必须相同，这限制了其使用范围。
## Where:
1.隔离复杂对象的创建和使用，相同的方法，不同执行顺序，产生不同事件结果
2.需要生成的产品对象的属性相互依赖，需要指定其生成顺序。

## How:

在学习使用建造者模式之前，我们需要了解几个概念。

#### Product：具体的产品。
#### Builder：抽象的建造方法。
#### ConcreteBuilder：具体的建造者。
#### Director(指挥官)：调用具体构造者创建的产品对象，负责将客户端传来指令交给具体的建造者。

![建造者模式UML](https://raw.githubusercontent.com/MuggleLee/PicGo/master/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E5%BB%BA%E9%80%A0%E8%80%85%E6%A8%A1%E5%BC%8F/Pattern-Builder.png)

主要有以下两种方式设计建造者模式：

## 一、通过Product，Builder，ConcreteBuilder，Director设计的建造者模式

ComputerElement类：具体的对象
```java
/**
 * 计算机元件
 */
public class ComputerElement {
    private String CPU;
    private String mainboard;
    private String memory;
    private String SSD;
    private String power;
    private String computerCase;

    public ComputerElement() {
    }

    public ComputerElement(String CPU, String mainboard, String memory, String SSD, String power, String computerCase) {
        this.CPU = CPU;
        this.mainboard = mainboard;
        this.memory = memory;
        this.SSD = SSD;
        this.power = power;
        this.computerCase = computerCase;
    }

    @Override
    public String toString() {
        final StringBuilder sb = new StringBuilder("电脑组装元件如下：\n");
        sb.append("CPU：")
                .append(CPU).append('\n');
        sb.append("主板:")
                .append(mainboard).append('\n');
        sb.append("内存：")
                .append(memory).append('\n');
        sb.append("SSD：")
                .append(SSD).append('\n');
        sb.append("电源：")
                .append(power).append('\n');
        sb.append("机箱：")
                .append(computerCase).append('\n');
        sb.append("正在组装中...").append('\n');
        sb.append("组装完成！").append('\n');
        return sb.toString();
    }

    // 省略get、set方法
    
}
```
ComputerBuilder：抽象的建造方法
```java
/**
 * 组装电脑抽象建造者
 */
public interface ComputerBuilder {
    ComputerBuilder buildCPU(String CPU);

    ComputerBuilder buildMainboard(String mainboard);

    ComputerBuilder buildMemory(String memory);

    ComputerBuilder buildSSD(String SSD);

    ComputerBuilder buildPower(String power);

    ComputerBuilder buildComputerCase(String computerCase);

    ComputerElement build();
}
```
ComputerActualBuilder：具体的建造者
```java
/**
 * 组装电脑具体建造者
 */
public class ComputerActualBuilder implements ComputerBuilder {

    ComputerElement computerElement = new ComputerElement();

    @Override
    public ComputerBuilder buildCPU(String CPU) {
        computerElement.setCPU(CPU);
        return this;
    }

    @Override
    public ComputerBuilder buildMainboard(String mainboard) {
        computerElement.setMainboard(mainboard);
        return this;
    }

    @Override
    public ComputerBuilder buildMemory(String memory) {
        computerElement.setMemory(memory);
        return this;
    }

    @Override
    public ComputerBuilder buildSSD(String SSD) {
        computerElement.setSSD(SSD);
        return this;
    }

    @Override
    public ComputerBuilder buildPower(String power) {
        computerElement.setPower(power);
        return this;
    }

    @Override
    public ComputerBuilder buildComputerCase(String computerCase) {
        computerElement.setComputerCase(computerCase);
        return this;
    }

    @Override
    public ComputerElement build() {
        return computerElement;
    }
}
```
ComputerDirector：指挥官
```java
/**
 * 指挥官
 */
public class ComputerDirector {

    private ComputerBuilder computerBuilder = new ComputerActualBuilder();

    public ComputerElement build(String CPU, String mainboard, String memory, String SSD, String power, String computerCase) {
        return computerBuilder.buildMainboard(mainboard)
                .buildCPU(CPU)
                .buildSSD(SSD)
                .buildMemory(memory)
                .buildPower(power)
                .buildComputerCase(computerCase)
                .build();
    }
}
```
Test：测试方法
```java
public class Test {
    public static void main(String[] args) {
        ComputerDirector director = new ComputerDirector();
        System.out.println(director.build("Intel酷睿六核处理器i5-8400","Intel B360","16G","1T","美商海盗船","机箱"));
    }
}
```
输出结果：
```java
电脑组装元件如下：
CPU：Intel酷睿六核处理器i5-8400
主板:Intel B360
内存：16G
SSD：1T
电源：美商海盗船
机箱：普通机箱
正在组装中...
组装完成！
```

## 二、通过静态内部类方式设计的建造者模式：

```java
public class ComputerDIY {

    private String CPU;
    private String mainboard;
    private String memory;
    private String SSD;
    private String power;
    private String computerCase;

    public ComputerDIY(ComputerBuilder computerBuilder) {
        this.CPU = computerBuilder.CPU;
        this.mainboard = computerBuilder.mainboard;
        this.memory = computerBuilder.memory;
        this.SSD = computerBuilder.SSD;
        this.power = computerBuilder.power;
        this.computerCase = computerBuilder.computerCase;
    }

    public String diy(){
        final StringBuilder sb = new StringBuilder("电脑配置如下：\n");
        sb.append("     CPU：")
                .append(CPU).append('\n');
        sb.append("     主板:")
                .append(mainboard).append('\n');
        sb.append("     内存：")
                .append(memory).append('\n');
        sb.append("     SSD：")
                .append(SSD).append('\n');
        sb.append("     电源：")
                .append(power).append('\n');
        sb.append("     机箱：")
                .append(computerCase).append('\n');
        sb.append("正在组装中...").append('\n');
        sb.append("组装完成！").append('\n');
        return sb.toString();
    }

    @Override
    public String toString() {
        final StringBuilder sb = new StringBuilder("{");
        sb.append("\"CPU\":\"")
                .append(CPU).append('\"');
        sb.append(",\"mainboard\":\"")
                .append(mainboard).append('\"');
        sb.append(",\"memory\":\"")
                .append(memory).append('\"');
        sb.append(",\"SSD\":\"")
                .append(SSD).append('\"');
        sb.append(",\"power\":\"")
                .append(power).append('\"');
        sb.append('}');
        return sb.toString();
    }

    public static class ComputerBuilder{
        private String CPU;
        private String mainboard;
        private String memory;
        private String SSD;
        private String power;
        private String computerCase;

        public ComputerBuilder buildCPU(String CPU) {
            this.CPU = CPU;
            return this;
        }

        public ComputerBuilder buildMainboard(String mainboard) {
            this.mainboard = mainboard;
            return this;
        }

        public ComputerBuilder buildMemory(String memory) {
            this.memory = memory;
            return this;
        }

        public ComputerBuilder buildSSD(String SSD) {
            this.SSD = SSD;
            return this;
        }

        public ComputerBuilder buildPower(String power) {
            this.power = power;
            return this;
        }

        public ComputerBuilder buildComputerCase(String computerCase) {
            this.computerCase = computerCase;
            return this;
        }


        public ComputerDIY build() {
            return new ComputerDIY(this);
        }
    }
}

```
```java
public class Test {
    public static void main(String[] args) {
        ComputerDIY computerDIY = new ComputerDIY.ComputerBuilder()
                .buildCPU("Intel酷睿六核处理器i5-8400")
                .buildMainboard("Intel B360")
                .buildMemory("16G")
                .buildSSD("16G")
                .buildPower("美商海盗船")
                .buildComputerCase("普通机箱")
                .build();
        System.out.println(computerDIY.diy());
    }
}
```
输出结果：
```java
电脑配置如下：
     CPU：Intel酷睿六核处理器i5-8400
     主板:Intel B360
     内存：16G
     SSD：16G
     电源：美商海盗船
     机箱：普通机箱
正在组装中...
组装完成！
```




# 总结
1. 建造者模式适合于创建的对象较复杂，由多个部件构成，各部件面临着复杂的变化，但构件间的建造顺序是稳定的。
2. 创建复杂对象的算法独立于该对象的组成部分以及它们的装配方式，即产品的构建过程和最终的表示是独立的。