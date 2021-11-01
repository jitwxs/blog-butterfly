---
title: Java 设计模式——工厂模式
categories: 
  - Java
  - 设计模式
abbrlink: 4caa56ee
date: 2018-09-26 19:15:21
references:
  - name: 工厂模式 - 菜鸟教程
    url: http://www.runoob.com/design-pattern/factory-pattern.html
  - name: JAVA设计模式之工厂模式—Factory Pattern
    url: https://www.cnblogs.com/carryjack/p/7709861.html
  - name: 三种工厂模式的使用选择
    url: https://blog.csdn.net/qq_17242957/article/details/52766885
---

## 一、什么是工厂模式

`工厂模式（Factory Pattern）`是 Java 中最常用的设计模式之一，它提供了一种创建对象的最佳方式。

在工厂模式中，我们**在创建对象时不会对客户端暴露创建逻辑，并且是通过使用一个共同的接口来指向新创建的对象**。

**优点：** 

1. 一个调用者想创建一个对象，只要知道其名称就可以了。 

2. 扩展性高，如果想增加一个产品，只要扩展一个工厂类就可以。

3. 屏蔽产品的具体实现，调用者只关心产品的接口。屏蔽产品的具体实现，调用者只关心产品的接口。

**缺点：**

每次增加一个产品时，都需要增加一个具体类和对象实现工厂，使得系统中类的个数成倍增加，在一定程度上增加了系统的复杂度，同时也增加了系统具体类的依赖。

工厂模式分为三种：`简单工厂模式`、`工厂方法模式`和`抽象工厂模式`。

这里以制造coffee的例子开始工厂模式设计之旅。我们知道coffee只是一种泛举，在点购咖啡时需要指定具体的咖啡种类：美式咖啡、卡布奇诺、拿铁等等。

```java
abstract class Coffee {
    public abstract String getName();
}

class Americano extends Coffee {
    @Override
    public String getName() {
        return "美式咖啡";
    }
}

class Cappuccino extends Coffee {
    @Override
    public String getName() {
        return "卡布奇诺";
    }

}

class Latte extends Coffee {
    @Override
    public String getName() {
        return "拿铁";
    }
}
```

## 二、简单工厂模式

简单工厂实际不能算作一种设计模式，它引入了创建者的概念，**将实例化的代码从应用代码中抽离，在创建者类的静态方法中只处理创建对象的细节，后续创建的实例如需改变，只需改造创建者类即可**，但由于使用静态方法来获取对象，使其不能在运行期间通过不同方式去动态改变创建行为，因此存在一定局限性。

```java
class SimpleFactory {
    public static Coffee createInstance(String type) {
        if ("americano".equals(type)) {
            return new Americano();
        } else if ("cappuccino".equals(type)) {
            return new Cappuccino();
        } else if ("latte".equals(type)) {
            return new Latte();
        } else {
            throw new RuntimeException("type[" + type + "]类型不可识别，没有匹配到可实例化的对象！");
        }
    }
}

public class Demo {
    public static void main(String[] args) {
        Coffee latte = SimpleFactory.createInstance("latte");
        System.out.println("创建的咖啡实例为:" + latte.getName());
        Coffee cappuccino = SimpleFactory.createInstance("cappuccino");
        System.out.println("创建的咖啡实例为:" + cappuccino.getName());
    }
}

/* output
创建的咖啡实例为:拿铁
创建的咖啡实例为:卡布奇诺
*/
```

## 三、工厂方法模式

**定义了一个创建对象的接口，但由子类决定要实例化的类是哪一个，工厂方法让类把实例化推迟到了子类。**

场景延伸：不同地区咖啡工厂受制于环境、原料等因素的影响，制造出的咖啡种类有限。中国咖啡工厂仅能制造卡布奇诺、拿铁，而美国咖啡工厂仅能制造美式咖啡、拿铁。

```java
abstract class CoffeeFactory {
    public abstract Coffee[] createCoffee();
}

// 中国咖啡工厂
class ChinaCoffeeFactory extends CoffeeFactory {
    @Override
    public Coffee[] createCoffee() {
        return new Coffee[]{new Cappuccino(), new Latte()};
    }
}

// 美国咖啡工厂
class AmericaCoffeeFactory extends CoffeeFactory {
    @Override
    public Coffee[] createCoffee() {
        return new Coffee[]{new Americano(), new Latte()};
    }
}

public class Demo {
    public static void main(String[] args) {
        CoffeeFactory chinaCoffeeFactory = new ChinaCoffeeFactory();
        Coffee[] chinaCoffees = chinaCoffeeFactory.createCoffee();
        System.out.println("中国咖啡工厂可以生产的咖啡有：");
        print(chinaCoffees);

        CoffeeFactory americaCoffeeFactory = new AmericaCoffeeFactory();
        Coffee[] americaCoffees = americaCoffeeFactory.createCoffee();
        System.out.println("美国咖啡工厂可以生产的咖啡有：");
        print(americaCoffees);
    }

    static void print(Coffee[] c) {
        for (Coffee coffee : c) {
            System.out.println(coffee.getName());
        }
    }
}

/* output
中国咖啡工厂可以生产的咖啡有：
卡布奇诺
拿铁
美国咖啡工厂可以生产的咖啡有：
美式咖啡
拿铁
*/
```

## 四、抽象工厂模式

**提供一个接口，用于创建相关或依赖对象的家族，而不需要明确指定具体类。**

在上述的场景上继续延伸：咖啡工厂做大做强，引入了新的饮品种类：茶、 碳酸饮料。中国工厂只能制造咖啡和茶，美国工厂只能制造咖啡和碳酸饮料。

如果用上述工厂方法方式，除去对应的产品实体类还需要新增2个抽象工厂（茶制造工厂、碳酸饮料制造工厂），4个具体工厂实现。随着产品的增多，会导致类爆炸。

所以这里引出一个概念产品家族，在此例子中，不同的饮品就组成我们的饮品家族， 饮品家族开始承担创建者的责任，负责制造不同的产品。

```java
abstract class Drink {
    public abstract String getName();
}

abstract class Coffee extends Drink {}

abstract class Tea extends Drink {}

abstract class Sodas extends Drink {}

class Latte extends Coffee {
    @Override
    public String getName() {
        return "拿铁";
    }
}

class MilkTea extends Tea {
    @Override
    public String getName() {
        return "奶茶";
    }
}

class CocaCola extends Sodas {
    @Override
    public String getName() {
        return "可口可乐";
    }
}

interface AbstractDrinksFactory {
    Coffee createCoffee();

    Tea createTea();

    Sodas createSodas();
}

class ChinaDrinksFactory implements AbstractDrinksFactory {
    @Override
    public Coffee createCoffee() {
        return new Latte();
    }

    @Override
    public Tea createTea() {
        return new MilkTea();
    }

    @Override
    public Sodas createSodas() {
        return null;
    }
}

class AmericaDrinksFactory implements AbstractDrinksFactory {
    @Override
    public Coffee createCoffee() {
        return new Latte();
    }

    @Override
    public Tea createTea() {
        return null;
    }

    @Override
    public Sodas createSodas() {
        return new CocaCola();
    }
}

public class Demo {
    public static void main(String[] args) {
        AbstractDrinksFactory chinaDrinksFactory = new ChinaDrinksFactory();
        Coffee coffee = chinaDrinksFactory.createCoffee();
        Tea tea = chinaDrinksFactory.createTea();
        Sodas sodas = chinaDrinksFactory.createSodas();
        System.out.println("中国饮品工厂有如下产品：");
        print(coffee);
        print(tea);
        print(sodas);

        AbstractDrinksFactory americaDrinksFactory = new AmericaDrinksFactory();
        coffee = americaDrinksFactory.createCoffee();
        tea = americaDrinksFactory.createTea();
        sodas = americaDrinksFactory.createSodas();
        System.out.println("美国饮品工厂有如下产品：");
        print(coffee);
        print(tea);
        print(sodas);
    }

    static void print(Drink drink) {
        if (drink == null) {
            System.out.println("产品：--");
        } else {
            System.out.println("产品：" + drink.getName());
        }
    }
}

/* output
中国饮品工厂有如下产品：
产品：拿铁
产品：奶茶
产品：--
美国饮品工厂有如下产品：
产品：拿铁
产品：--
产品：可口可乐
*/
```

## 五、模式对比

如果产品单一，最合适用工厂模式，但是如果有多个业务品种、业务分类时，通过抽象工厂模式产生需要的对象是一种非常好的解决方式。

| 工厂方法模式                               | 抽象工厂模式                               |
| :----------------------------------------- | :----------------------------------------- |
| 针对的是一个产品等级结构                   | 针对的是面向多个产品等级结构               |
| 一个抽象产品类                             | 多个抽象产品类                             |
| 可以派生出多个具体产品类                   | 每个抽象产品类可以派生出多个具体产品类     |
| 一个抽象工厂类，可以派生出多个具体工厂类   | 一个抽象工厂类，可以派生出多个具体工厂类   |
| 每个具体工厂类只能创建一个具体产品类的实例 | 每个具体工厂类可以创建多个具体产品类的实例 |
