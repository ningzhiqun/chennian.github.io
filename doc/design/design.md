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

#### 代理模式

个人觉得代理模式是必须需要掌握的。代理模式可以帮助我们屏蔽底层实现类的实现细节，在实现前后添加一部分逻辑。即通过代理对象访问目标对象，这样可以在不修改原目标对象的前提下，提供额外的功能操作，扩展目标对象的功能。

> 其实，理解了“代理”这个词的意思，其实这个模式也就理解了。

```java
public interface FoodService {
    Food makeChicken();
    Food makeNoodle();
}

public class FoodServiceImpl implements FoodService {
    public Food makeChicken() {
          Food f = new Chicken()
        f.setChicken("1kg");
          f.setSpicy("1g");
          f.setSalt("3g");
        return f;
    }
    public Food makeNoodle() {
        Food f = new Noodle();
        f.setNoodle("500g");
        f.setSalt("5g");
        return f;
    }
}

// 代理要表现得“就像是”真实实现类，所以需要实现 FoodService
public class FoodServiceProxy implements FoodService {

    // 内部一定要有一个真实的实现类，当然也可以通过构造方法注入
    private FoodService foodService = new FoodServiceImpl();

    public Food makeChicken() {
        System.out.println("我们马上要开始制作鸡肉了");

        // 如果我们定义这句为核心代码的话，那么，核心代码是真实实现类做的，
        // 代理只是在核心代码前后做些“无足轻重”的事情
        Food food = foodService.makeChicken();

        System.out.println("鸡肉制作完成啦，加点胡椒粉"); // 增强
          food.addCondiment("pepper");

        return food;
    }
    public Food makeNoodle() {
        System.out.println("准备制作拉面~");
        Food food = foodService.makeNoodle();
        System.out.println("制作完成啦")
        return food;
    }
}
```

客户端调用，注意，我们要用代理来实例化接口：

```java
// 这里用代理类来实例化
FoodService foodService = new FoodServiceProxy();
foodService.makeChicken();
```

代理模式本质上就是对既有方法不改动的情况下，对方法进行**“方法增强”**。在 `SPRING` 框架中，面向切面这一概念就是基于代理模式。在 spring 中我们不需要自己定义代理类，框架会帮我们动态代理我们的类。在spring 中的动态代理，又分为 JDK 动态代理（面向接口） 和 CGlib 代理。感兴趣的可以去网上查阅这部分的资料。

#### 享元模式

享元模式我更喜欢自己给他起的名字——缓存模式，即通过容器将创建好的对象缓存起来，之后每次用，就直接从容器中取出来就好了，不需要二次创建。在 java 的包装类中就能看见这样的代码（想必大家面试的时候可能被问过 Integer 是否相等的问题）：

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

享元模式就讲到这了。

#### 组合模式

其实组合模式最常见的实现，就是树这种数据结构了。组合模式用于表示具有层次结构的数据，使得我们**对单个对象和组合对象的访问具有一致性**。

上面的话就是，整体对象的每一个子节点有着相同的访问方式和使用方式。

直接看一个例子吧，每个员工都有姓名、部门、薪水这些属性，同时还有下属员工集合（虽然可能集合为空），而下属员工和自己的结构是一样的，也有姓名、部门这些属性，同时也有他们的下属员工集合。

```java
public class Employee {
   private String name;
   private String dept;
   private int salary;
   private List<Employee> subordinates; // 下属

   public Employee(String name,String dept, int sal) {
      this.name = name;
      this.dept = dept;
      this.salary = sal;
      subordinates = new ArrayList<Employee>();
   }

   public void add(Employee e) {
      subordinates.add(e);
   }

   public void remove(Employee e) {
      subordinates.remove(e);
   }

   public List<Employee> getSubordinates(){
     return subordinates;
   }

   public String toString(){
      return ("Employee :[ Name : " + name + ", dept : " + dept + ", salary :" + salary+" ]");
   }   
}
```

#### 门面模式

门面模式（也叫外观模式，Facade Pattern）在许多源码中有使用，比如 slf4j 就可以理解为是门面模式的应用。这是一个简单的设计模式，我们直接上代码再说吧。

首先，我们定义一个接口：

```java
public interface Shape {
   void draw();
}
```

定义几个实现类：

```java
public class Circle implements Shape {
    @Override
    public void draw() {
       System.out.println("Circle::draw()");
    }
}

public class Rectangle implements Shape {
    @Override
    public void draw() {
       System.out.println("Rectangle::draw()");
    }
}
```

客户端调用：

```java
public static void main(String[] args) {
    // 画一个圆形
      Shape circle = new Circle();
      circle.draw();

      // 画一个长方形
      Shape rectangle = new Rectangle();
      rectangle.draw();
}
```

以上是我们常写的代码，我们需要画圆就要先实例化圆，画长方形就需要先实例化一个长方形，然后再调用相应的 draw() 方法。

下面，我们看看怎么用门面模式来让客户端调用更加友好一些。

我们先定义一个门面：

```java
public class ShapeMaker {
   private Shape circle;
   private Shape rectangle;
   private Shape square;

   public ShapeMaker() {
      circle = new Circle();
      rectangle = new Rectangle();
      square = new Square();
   }

  /**
   * 下面定义一堆方法，具体应该调用什么方法，由这个门面来决定
   */

   public void drawCircle(){
      circle.draw();
   }
   public void drawRectangle(){
      rectangle.draw();
   }
   public void drawSquare(){
      square.draw();
   }
}
```

看看现在客户端怎么调用：

```java
public static void main(String[] args) {
  ShapeMaker shapeMaker = new ShapeMaker();

  // 客户端调用现在更加清晰了
  shapeMaker.drawCircle();
  shapeMaker.drawRectangle();
  shapeMaker.drawSquare();        
}
```

门面模式的优点显而易见，客户端不再需要关注实例化时应该使用哪个实现类，直接调用门面提供的方法就可以了，因为门面类提供的方法的方法名对于客户端来说已经很友好了。

#### 门面模式（未完成）

#### 桥梁模式（未完成）

#### 适配器模式（未完成）

这里就不给大家说了，是在分不清和代理模式有什么区别，这里就不给大家误导了。

------



### 行为型模式

行为型模式关注的是各个类之间的相互作用，将职责划分清楚，使得我们的代码更加地清晰。

#### 策略模式

策略模式太常用了，所以把它放到最前面进行介绍。它比较简单，我就不废话，直接用代码说事吧。

下面设计的场景是，我们需要画一个图形，可选的策略就是用红色笔来画，还是绿色笔来画，或者蓝色笔来画。

首先，先定义一个策略接口：

```java
public interface Strategy {
   public void draw(int radius, int x, int y);
}
```

然后我们定义具体的几个策略：

```java
public class RedPen implements Strategy {
   @Override
   public void draw(int radius, int x, int y) {
      System.out.println("用红色笔画图，radius:" + radius + ", x:" + x + ", y:" + y);
   }
}
public class GreenPen implements Strategy {
   @Override
   public void draw(int radius, int x, int y) {
      System.out.println("用绿色笔画图，radius:" + radius + ", x:" + x + ", y:" + y);
   }
}
public class BluePen implements Strategy {
   @Override
   public void draw(int radius, int x, int y) {
      System.out.println("用蓝色笔画图，radius:" + radius + ", x:" + x + ", y:" + y);
   }
}
```

使用策略的类：

```java
public class Context {
   private Strategy strategy;

   public Context(Strategy strategy){
      this.strategy = strategy;
   }

   public int executeDraw(int radius, int x, int y){
      return strategy.draw(radius, x, y);
   }
}
```

客户端演示：

```java
public static void main(String[] args) {
    Context context = new Context(new BluePen()); // 使用绿色笔来画
      context.executeDraw(10, 0, 0);
}
```



#### 观察者模式

观察者模式对于我们来说，真是再简单不过了。无外乎两个操作，观察者订阅自己关心的主题和主题有数据变化后通知观察者们。

首先，需要定义主题，每个主题需要持有观察者列表的引用，用于在数据变更的时候通知各个观察者：

```java
public class Subject {
    private List<Observer> observers = new ArrayList<Observer>();
    private int state;
    public int getState() {
        return state;
    }
    public void setState(int state) {
        this.state = state;
        // 数据已变更，通知观察者们
        notifyAllObservers();
    }
    // 注册观察者
    public void attach(Observer observer) {
        observers.add(observer);
    }
    // 通知观察者们
    public void notifyAllObservers() {
        for (Observer observer : observers) {
            observer.update();
        }
    }
}
```

定义观察者接口：

```java
public abstract class Observer {
    protected Subject subject;
    public abstract void update();
}
```

其实如果只有一个观察者类的话，接口都不用定义了，不过，通常场景下，既然用到了观察者模式，我们就是希望一个事件出来了，会有多个不同的类需要处理相应的信息。比如，订单修改成功事件，我们希望发短信的类得到通知、发邮件的类得到通知、处理物流信息的类得到通知等。

我们来定义具体的几个观察者类：

```java
public class BinaryObserver extends Observer {
    // 在构造方法中进行订阅主题
    public BinaryObserver(Subject subject) {
        this.subject = subject;
        // 通常在构造方法中将 this 发布出去的操作一定要小心
        this.subject.attach(this);
    }
    // 该方法由主题类在数据变更的时候进行调用
    @Override
    public void update() {
        String result = Integer.toBinaryString(subject.getState());
        System.out.println("订阅的数据发生变化，新的数据处理为二进制值为：" + result);
    }
}

public class HexaObserver extends Observer {
    public HexaObserver(Subject subject) {
        this.subject = subject;
        this.subject.attach(this);
    }
    @Override
    public void update() {
        String result = Integer.toHexString(subject.getState()).toUpperCase();
        System.out.println("订阅的数据发生变化，新的数据处理为十六进制值为：" + result);
    }
}
```

客户端使用也非常简单：

```java
public static void main(String[] args) {
    // 先定义一个主题
    Subject subject1 = new Subject();
    // 定义观察者
    new BinaryObserver(subject1);
    new HexaObserver(subject1);

    // 模拟数据变更，这个时候，观察者们的 update 方法将会被调用
    subject.setState(11);
}
```

output:

```
订阅的数据发生变化，新的数据处理为二进制值为：1011
订阅的数据发生变化，新的数据处理为十六进制值为：B
```

当然，jdk 也提供了相似的支持，具体的大家可以参考 java.util.Observable 和 java.util.Observer 这两个类。

实际生产过程中，观察者模式往往用消息中间件来实现，如果要实现单机观察者模式，笔者建议读者使用 Guava 中的 EventBus，它有同步实现也有异步实现，本文主要介绍设计模式，就不展开说了。

还有，即使是上面的这个代码，也会有很多变种，大家只要记住核心的部分，那就是一定有一个地方存放了所有的观察者，然后在事件发生的时候，遍历观察者，调用它们的回调函数。



#### 责任链模式

责任链通常需要先建立一个单向链表，然后调用方只需要调用头部节点就可以了，后面会自动流转下去。比如流程审批就是一个很好的例子，只要终端用户提交申请，根据申请的内容信息，自动建立一条责任链，然后就可以开始流转了。

有这么一个场景，用户参加一个活动可以领取奖品，但是活动需要进行很多的规则校验然后才能放行，比如首先需要校验用户是否是新用户、今日参与人数是否有限额、全场参与人数是否有限额等等。设定的规则都通过后，才能让用户领走奖品。

> 如果产品给你这个需求的话，我想大部分人一开始肯定想的就是，用一个 List 来存放所有的规则，然后 foreach 执行一下每个规则就好了。不过，读者也先别急，看看责任链模式和我们说的这个有什么不一样？

首先，我们要定义流程上节点的基类：

```java
public abstract class RuleHandler {
    // 后继节点
    protected RuleHandler successor;

    public abstract void apply(Context context);

    public void setSuccessor(RuleHandler successor) {
        this.successor = successor;
    }

    public RuleHandler getSuccessor() {
        return successor;
    }
}
```

接下来，我们需要定义具体的每个节点了。

校验用户是否是新用户：

```java
public class NewUserRuleHandler extends RuleHandler {
    public void apply(Context context) {
        if (context.isNewUser()) {
            // 如果有后继节点的话，传递下去
            if (this.getSuccessor() != null) {
                this.getSuccessor().apply(context);
            }
        } else {
            throw new RuntimeException("该活动仅限新用户参与");
        }
    }
}
```

校验用户所在地区是否可以参与：

```java
public class LocationRuleHandler extends RuleHandler {
    public void apply(Context context) {
        boolean allowed = activityService.isSupportedLocation(context.getLocation);
        if (allowed) {
            if (this.getSuccessor() != null) {
                this.getSuccessor().apply(context);
            }
        } else {
            throw new RuntimeException("非常抱歉，您所在的地区无法参与本次活动");
        }
    }
}
```

校验奖品是否已领完：

```java
public class LimitRuleHandler extends RuleHandler {
    public void apply(Context context) {
        int remainedTimes = activityService.queryRemainedTimes(context); // 查询剩余奖品
        if (remainedTimes > 0) {
            if (this.getSuccessor() != null) {
                this.getSuccessor().apply(userInfo);
            }
        } else {
            throw new RuntimeException("您来得太晚了，奖品被领完了");
        }
    }
}
```

客户端：

```java
public static void main(String[] args) {
    RuleHandler newUserHandler = new NewUserRuleHandler();
    RuleHandler locationHandler = new LocationRuleHandler();
    RuleHandler limitHandler = new LimitRuleHandler();

    // 假设本次活动仅校验地区和奖品数量，不校验新老用户
    locationHandler.setSuccessor(limitHandler);

    locationHandler.apply(context);
}
```

代码其实很简单，就是先定义好一个链表，然后在通过任意一节点后，如果此节点有后继节点，那么传递下去。

至于它和我们前面说的用一个 List 存放需要执行的规则的做法有什么异同，留给读者自己琢磨吧。



#### 模板方法模式

在含有继承结构的代码中，模板方法模式是非常常用的。

通常会有一个抽象类：

```java
public abstract class AbstractTemplate {
    // 这就是模板方法
    public void templateMethod() {
        init();
        apply(); // 这个是重点
        end(); // 可以作为钩子方法
    }

    protected void init() {
        System.out.println("init 抽象层已经实现，子类也可以选择覆写");
    }

    // 留给子类实现
    protected abstract void apply();

    protected void end() {
    }
}
```

模板方法中调用了 3 个方法，其中 apply() 是抽象方法，子类必须实现它，其实模板方法中有几个抽象方法完全是自由的，我们也可以将三个方法都设置为抽象方法，让子类来实现。也就是说，模板方法只负责定义第一步应该要做什么，第二步应该做什么，第三步应该做什么，至于怎么做，由子类来实现。

我们写一个实现类：

```java
public class ConcreteTemplate extends AbstractTemplate {
    public void apply() {
        System.out.println("子类实现抽象方法 apply");
    }

    public void end() {
        System.out.println("我们可以把 method3 当做钩子方法来使用，需要的时候覆写就可以了");
    }
}
```

客户端调用演示：

```java
public static void main(String[] args) {
    AbstractTemplate t = new ConcreteTemplate();
    // 调用模板方法
    t.templateMethod();
}
```

代码其实很简单，基本上看到就懂了，关键是要学会用到自己的代码中。



#### 状态模式

废话我就不说了，我们说一个简单的例子。商品库存中心有个最基本的需求是减库存和补库存，我们看看怎么用状态模式来写。

核心在于，我们的关注点不再是 Context 是该进行哪种操作，而是关注在这个 Context 会有哪些操作。

定义状态接口：

```java
public interface State {
    public void doAction(Context context);
}
```

定义减库存的状态：

```java
public class DeductState implements State {

    public void doAction(Context context) {
        System.out.println("商品卖出，准备减库存");
        context.setState(this);

        //... 执行减库存的具体操作
    }

    public String toString() {
        return "Deduct State";
    }
} 
```

定义补库存状态：

```java
public class RevertState implements State {

    public void doAction(Context context) {
        System.out.println("给此商品补库存");
        context.setState(this);

        //... 执行加库存的具体操作
    }

    public String toString() {
        return "Revert State";
    }
}
```

前面用到了 context.setState(this)，我们来看看怎么定义 Context 类：

```java
public class Context {
    private State state;
      private String name;
      public Context(String name) {
        this.name = name;
    }

      public void setState(State state) {
        this.state = state;
    }
      public void getState() {
        return this.state;
    }
}
```

我们来看下客户端调用，大家就一清二楚了：

```java
public static void main(String[] args) {
    // 我们需要操作的是 iPhone X
    Context context = new Context("iPhone X");

    // 看看怎么进行补库存操作
      State revertState = new RevertState();
      revertState.doAction(context);

    // 同样的，减库存操作也非常简单
      State deductState = new DeductState();
      deductState.doAction(context);

      // 如果需要我们可以获取当前的状态
    // context.getState().toString();
}
```

读者可能会发现，在上面这个例子中，如果我们不关心当前 context 处于什么状态，那么 Context 就可以不用维护 state 属性了，那样代码会简单很多。

不过，商品库存这个例子毕竟只是个例，我们还有很多实例是需要知道当前 context 处于什么状态的。

### 总结

学习设计模式的目的是为了让我们的代码更加的优雅、易维护、易扩展。其实觉得一篇文章写下来收货最大的是自己。哈哈，我往大家有空也可以参考着网上文档，结合着自己的经验写一下，收获满满。
