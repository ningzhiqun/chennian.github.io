记得刚开始工作的时候，看到网上说设计模式，每次看的都是云里雾里的样子。现在工作3年多了，现在回过头来看设计模式，突然有种顿悟的感觉。（ps：作为一个过来人的经验告诉大家，如果你刚工作不久，现在看了设计模式，可能不是很理解，等你积累了一定的工作经验的时候，回过头来看，会觉得 so easy ！！！）。好了，废话不多说，接下来直奔主题。

设计模式本质上是前人工作中，对自己的代码进行的抽象总结，其中被大家广泛认同的是 Gang of Four（GOF）的分类了。他们将设计模式分为23中经典的模式。根据用途我们有可以将其分为三大类：创建型模式、结构式模式和行为模式。（大家要相信，即使你不看，如果一直从事程序猿工作，会在日常工作中用到某些模式）

**创建者模式比较容易理解，如果你是一位从事 Java web 开发，spring core 就建立在这上面**。

------



### 创建型模式

创建型模式关注的核心目标就是如何创建一个对象，即关注的核心是类的创建过程。我们在日常开发中会经常面对大量对象的创建或者复杂对象的创建，而创建型模式就是帮助我们解决这种问题的。

创建型模式分为：1、工厂方法；2、抽象工程；3、建造者；4、原型模式；5、单例模式。

#### 简单工厂模式

这个很简单，简单都不知道怎么描述，先上代码：

```java
public class FruitsFactory {

    public static Fruit getFruit(String name) {
      	// Apple 和 Banana 都继承了 Fruit 接口
        switch (name){
            case "apple":
            		retrun new Apple();
          	case "banana"：
              	return new Banana();
            default:
            		return null;
        }
    }
}
```

这样就能清楚的看到，我们只需要定义一个xxxFactory 工厂类，里面有一个方法，根据我们的需要去获取不同实现类。

> 我们强调职责单一原则，一个类只提供一种功能，FoodFactory 的功能就是只要负责生产各种 Food。

#### 工程方法模式

简单工厂模式是简单，但是很多情况下它都有着它的局限性，比如常常我们需要两个或者两个以上的工厂。这个时候也就引申出我们的——工程方法模式。

```java
public interface FruitsFactory {
    Fruit getFruit(String name);
}
public class ChineseFruitFactory implements FruitsFactory {

    @Override
    public Fruit getFruit(String name) {
      	// ChineseApple 和 ChineseBanana 都继承了 Fruit 接口
        switch (name){
            case "apple":
            		retrun new ChineseApple();
          	case "banana"：
              	return new ChineseBanana();
            default:
            		return null;
        }
    }
}
public class AmericanFruitFactory implements FruitsFactory {

    @Override
    public Fruit getFruit(String name) {
      	// AmericanApple 和 AmericanBanana 都继承了 Fruit 接口
        switch (name){
            case "apple":
            		retrun new AmericanApple();
          	case "banana"：
              	return new AmericanBanana();
            default:
            		return null;
        }
    }
}
```

虽然看起来两个工厂很像，但是，不同的工厂生产的东西是不一样的。相较于简单工厂模式，工厂方法模式多了一步选择合适工厂的步骤。

**工厂方法模式的核心在于选择合适的工厂**。比如，Java 核心库提供了标准 JDBC 的接口，但是它的实现有 MySQL 、SQL server 等实现，分别将数据写入到不同的数据库中。因此在我们日常的开发 springboot 应用中，第一步就是选择合适的数据库连接。

#### 抽象工厂模型

大家都知道，一台可供程序猿勉强使用的电脑，必定有 CPU、主板、显示器等部件。

因为电脑是由许多的构件组成的，我们将 CPU 和主板进行抽象，然后 CPU 由 CPUFactory 生产，主板由 MainBoardFactory 生产，然后，我们再将 CPU 和主板搭配起来组合在一起，如下图：

![factory-1](https://www.javadoop.com/blogimages/design-pattern/abstract-factory-1.png)

这个时候的客户端调用是这样的：

```java
// 得到 Intel 的 CPU
CPUFactory cpuFactory = new IntelCPUFactory();
CPU cpu = intelCPUFactory.makeCPU();

// 得到 AMD 的主板
MainBoardFactory mainBoardFactory = new AmdMainBoardFactory();
MainBoard mainBoard = mainBoardFactory.make();

// 组装 CPU 和主板
Computer computer = new Computer(cpu, mainBoard);
```

单独看 CPU 工厂和主板工厂，它们分别是前面我们说的**工厂方法模式**。这种方式也容易扩展，因为要给电脑加硬盘的话，只需要加一个 HardDiskFactory 和相应的实现即可，不需要修改现有的工厂。

但是，现实生活中存在一个问题，**Intel 和 AMD 两家厂商有着商业竞争关系，它们就不想彼此的主板能相互使用**。那么这代码就容易出错，因为客户端并不知道它们不兼容，也就会错误地出现随意组合。

因此这个时候，我们不再定义 CPU 工厂、主板工厂、硬盘工厂、显示屏工厂等等，我们直接定义电脑工厂，每个电脑工厂负责生产所有的设备，这样能保证肯定不存在兼容问题。

![abstract-factory-3](https://www.javadoop.com/blogimages/design-pattern/abstract-factory-3.png)

这个时候，对于客户端来说，不再需要单独挑选 CPU厂商、主板厂商、硬盘厂商等，直接选择一家品牌工厂，品牌工厂会负责生产所有的东西，而且能保证肯定是兼容可用的。

```java
public static void main(String[] args) {
    // 第一步就要选定一个“大厂”
    ComputerFactory cf = new AmdFactory();
    // 从这个大厂造 CPU
    CPU cpu = cf.makeCPU();
    // 从这个大厂造主板
    MainBoard board = cf.makeMainBoard();
    // 从这个大厂造硬盘
    HardDisk hardDisk = cf.makeHardDisk();

    // 将同一个厂子出来的 CPU、主板、硬盘组装在一起
    Computer result = new Computer(cpu, board, hardDisk);
}
```

当然，抽象工厂的问题也是显而易见的，比如我们要加个显示器，就需要修改所有的工厂，给所有的工厂都加上制造显示器的方法。这有点违反了**对修改关闭，对扩展开放**这个设计原则。

#### 单例模式

单例模式的实现方式有很多种，每一种都有自己的特点。

**饿汉式：**

```java
public class Singleton {
    // 首先，将 new Singleton() 堵死
    private Singleton() {};
    // 创建私有静态实例，意味着这个类第一次使用的时候就会进行创建
    private static Singleton instance = new Singleton();

    public static Singleton getInstance() {
        return instance;
    }
    // 瞎写一个静态方法。这里想说的是，如果我们只是要调用 Singleton.getDate(...)，
    // 本来是不想要生成 Singleton 实例的，不过没办法，已经生成了
    public static Date getDate(String mode) {return new Date();}
}
```

> 看到网上很多人说饿汉式的缺点：你定义了一个单例的类，不需要其实例，可是你却把一个或几个你会用到的静态方法塞到这个类中。实际上本人觉得这都是可以规避的。另外，如果不想使用它，为什么要写这行代码呢？

**懒汉式：**

```java
public class Singleton {
    // 首先，也是先堵死 new Singleton() 这条路
    private Singleton() {}
    // 和饿汉模式相比，这边不需要先实例化出来，注意这里的 volatile，它是必须的
    private static volatile Singleton instance = null;
		
  	// 这个地方需要加锁，当然有两个加锁的地方
  	// 第一种：
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
  	// 第二种：
  	public static synchronized Singleton getInstance() {
        if (instance == null) {
             instance = new Singleton();
        }
        return instance;
    }
}
```

> 当然，synchronized 第二种比第一种性能差一点，毕竟每次获取时都要获取锁。
>
> 最后，枚举类天生为单例模式，用 JVM 保证为单例。

#### 建造模式

经常碰见的 XxxBuilder 的类，通常都是建造者模式的产物。建造者模式其实有很多的变种，但是对于客户端来说，我们的使用通常都是一个模式的：

```java
Food food = new FoodBuilder().a().b().c().build();
Food food = Food.builder().a().b().c().build();
```

套路就是先 new 一个 Builder，然后可以链式地调用一堆方法，最后再调用一次 build() 方法，我们需要的对象就有了。

来一个中规中矩的建造者模式：

```java
class User {
    // 下面是“一堆”的属性
    private String name;
    private String password;
    private String nickName;
    private int age;

    // 构造方法私有化，不然客户端就会直接调用构造方法了
    private User(String name, String password, String nickName, int age) {
        this.name = name;
        this.password = password;
        this.nickName = nickName;
        this.age = age;
    }
    // 静态方法，用于生成一个 Builder，这个不一定要有，不过写这个方法是一个很好的习惯，
    // 有些代码要求别人写 new User.UserBuilder().a()...build() 看上去就没那么好
    public static UserBuilder builder() {
        return new UserBuilder();
    }

    public static class UserBuilder {
        // 下面是和 User 一模一样的一堆属性
        private String  name;
        private String password;
        private String nickName;
        private int age;

        private UserBuilder() {
        }

        // 链式调用设置各个属性值，返回 this，即 UserBuilder
        public UserBuilder name(String name) {
            this.name = name;
            return this;
        }

        public UserBuilder password(String password) {
            this.password = password;
            return this;
        }

        public UserBuilder nickName(String nickName) {
            this.nickName = nickName;
            return this;
        }

        public UserBuilder age(int age) {
            this.age = age;
            return this;
        }

        // build() 方法负责将 UserBuilder 中设置好的属性“复制”到 User 中。
        // 当然，可以在 “复制” 之前做点检验
        public User build() {
            if (name == null || password == null) {
                throw new RuntimeException("用户名和密码必填");
            }
            if (age <= 0 || age >= 150) {
                throw new RuntimeException("年龄不合法");
            }
            // 还可以做赋予”默认值“的功能
              if (nickName == null) {
                nickName = name;
            }
            return new User(name, password, nickName, age);
        }
    }
}
```

核心是：先把所有的属性都设置给 Builder，然后 build() 方法的时候，将这些属性**复制**给实际产生的对象。

看看客户端的调用：

```java
public class APP {
    public static void main(String[] args) {
        User d = User.builder()
                .name("foo")
                .password("pAss12345")
                .age(25)
                .build();
    }
}
```

说实话，建造者模式的**链式**写法很吸引人，但是，多写了很多“无用”的 builder 的代码，感觉这个模式没什么用。不过，当属性很多，而且有些必填，有些选填的时候，这个模式会使代码清晰很多。我们可以在 **Builder 的构造方法**中强制让调用者提供必填字段，还有，在 build() 方法中校验各个参数比在 User 的构造方法中校验，代码要优雅一些。

> 题外话，强烈建议读者使用 lombok，用了 lombok 以后，上面的一大堆代码会变成如下这样:

```java
@Builder
class User {
    private String  name;
    private String password;
    private String nickName;
    private int age;
}
```

> 怎么样，省下来的时间是不是又可以干点别的了。

当然，如果你只是想要链式写法，不想要建造者模式，有个很简单的办法，User 的 getter 方法不变，所有的 setter 方法都让其 **return this** 就可以了，然后就可以像下面这样调用：

```java
User user = new User().setName("").setPassword("").setAge(20);
```

#### 原型模式

原型模式对比到现实生活中就是克隆人差不多的概念。不光克隆外表，连内在都一模一样。

在 Java 中有浅拷贝和深拷贝两种，其实，原型模式的实现之一就是深拷贝。即基于现有的对象复制出来一个一样的对象。

Object 类中有一个 clone() 方法，它用于生成一个新的对象，当然，如果我们要调用这个方法，java 要求我们的类必须先**实现 Cloneable 接口**，此接口没有定义任何方法，但是不这么做的话，在 clone() 的时候，会抛出 CloneNotSupportedException 异常。

```java
protected native Object clone() throws CloneNotSupportedException;
```

原型模式就说这么多了，网上有很多变种说原型模型，**个人觉得其实本质上就是根据已有的对象复制出来一个地址不同的对象。**

------



### 结构型模式

前面创建型模式介绍了创建对象的一些设计模式，这节介绍的结构型模式旨在通过改变代码结构来达到解耦的目的，使得我们的代码容易维护和扩展。



loading ......(过段时间写)

