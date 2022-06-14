#### 一、Java文件结构（魔术、版本号、常量池等）

#### 二、Java常见数据结构
- list：ArrayList（数组） 、 LinkedList（链表）
- set：hashset 、LinkedHashSet、treeSet
- map： HashMap、ConcurrentHashMap

#### 三、Java内存分布
- 程序计数器、虚拟机栈、本地方法栈、堆、方法区（JDK1.7之前的永久代、JDK1.8之后的元空间）、直接内存
- 线程自有：程序计数器、虚拟机栈、本地方法栈

#### 四、Java内存模型（JMM）
在多线程代码中，哪些行为是正确的、合法的，以及多线程之间如何进行通信，代码中变量的读写行为如何反应到内存、CPU缓存的底层细节。
- 关键字：volatile、final和synchronized

#### 五、对象分布、对象访问
- 对象分布：对象头、实例数据、对齐数据
- 对象访问：指针、句柄

#### 六、Java对象创建过程
加载、分配内存（指针碰撞、空闲列表）、初始化零值、设置对象头、执行init

#### 七、Java类加载过程
遇到如下情况会进行类加载：
- 遇到new 、static 等字节码指令
- 使用反射调用的时候时发现类未加载
- 初始化子类时发现父类未加载
- 执行main的方法

#### 八、重载和重写
- 重载：类相同方法名的方法有多个
- 重写：子类对父类的方法覆盖
- **重写两同、两小、一大**

---

**加载顺序**
1. 加载（获取class文件的二进制流、在方法区生成元数据、在堆中生成class对象）
2. 连接
    2. 验证
    3. 准备
    4. 解析
3. 初始化
4. 使用
5. 卸载（堆类没有实例 、 没有任何地方引用 、类加载器被回收）

#### 八、Java初始化顺序
父类静态字段、父类静态代码块、子类静态字段、子类静态代码块
、父类变量、父类代码块、父类构造函数、子类变量、子类代码块、子类构造函数

#### 九、类加载器、双亲委派、破坏双亲委派
- 类加载器：BootstrapClassLoader 、ExtensionClassLoader 、AppClassLoader
- findClass ：不破坏   loadClass：破坏


---
**GC 相关**

#### 十、OOM
- StackOverFlowError 、OutOfMemoryError
- 除了程序计数器以外，所有地方都会发生oom

#### 十一、垃圾回收算法
- GC算法：可达性算法、计数引用法
- 三大算法：标记-清除、标记-整理、标记-复制
**ps：新生代常用复制算法（从各代垃圾特点说明）**

#### 十二、FULL GC
1. system.gc
2. class 无法加载到堆中
3. 大对象等进入老年代不足
4. 空间担保机制导致空间不足
5. CMS收集器导致空间不足

#### 十二、Java访问权限修饰符
- publish：所有都可以访问
- private：
- protect：派生类、同包
- default：派生类

#### 十三、四种引用
- 强引用：new 出来的对象都是
- 软引用：缓存
- 弱引用：threadlocal
- 虚引用：对象回收


---
** BQ 相关 **
#### 十四、AQS
- Node：双向队列
    - mode: shard 、exclusive
    - waitStatus：1 0 -1 -2 -3
    - prednode 、nextNode、waitNode
- head
- tail
- state: 大于0说明现在都有人占用锁
- lock流程（公平锁）：
    1. 检查等待队列中是否存在等待线程
    2. 通过cas 获取锁
    3. 如果上述过程没有获取锁，检查是否是重入锁
    4. 清除队列中的取消状态
    5. 重复上面的流程
- Condition （持有对象锁才能进行相应的操作）
- 常用类（ReentrantLock 、 CountDownLock、CyclicBarrier、Semaphore）

#### 十五、BlockingQueue
插入：报错、阻塞、特殊值、超时
- ArrayBlockingQueue：底层是数组，有界队列，如果我们要使用生产者-消费者模式，这是非常好的选择。

- LinkedBlockingQueue：底层是链表，可以当做无界和有界队列来使用，所以大家不要以为它就是无界队列。

- SynchronousQueue：本身不带有空间来存储任何元素，使用上可以选择公平模式和非公平模式。

- PriorityBlockingQueue：是无界队列，基于数组，数据结构为二叉堆，数组第一个也是树的根节点总是最小值。

#### 十六、atomic 原子类 、 atomic 集合类 、 atomic 引用类


---
io相关

#### 十七、IO 类型
同步 or 异步、 阻塞 or 非阻塞

#### 十八、IO 五种类型
- 同步阻塞
- 同步非阻塞
- 多路复用
- 信号
- 异步

#### 十九、NIO三大组件
Buffer、channel 、selector

#### 二十、netty（后面会写一篇基于源码的）
bootstrap、group、channel、optional、handle、childhandle


---
**线程池相关**

#### 二十一、线程池构造函数
coresize、maximumPoolSize、keepAliveTime、unit、workQueue、threadFactory、rejecthandler

#### 二十二、线程池的五种状态
RUNNING、SHUTDOWN、STOP、TIDYING、TERMINATED

#### 二十三、线程池excute 方法

```
int c = ctl.get();
if (workerCountOf(c) < corePoolSize) {
    if (addWorker(command, true))
        return;
    c = ctl.get();
}
if (isRunning(c) && workQueue.offer(command)) {
    int recheck = ctl.get();
    if (! isRunning(recheck) && remove(command))
        reject(command);
    else if (workerCountOf(recheck) == 0)
        addWorker(null, false);
}
else if (!addWorker(command, false))
    reject(command);
```

#### 二十四、线程的六种状态
new、ready、running、blocked、timeoutwait、waiting、Terminated


---
**事务**
**ACID：原子性、一致性、隔离性、持久性**

#### 二十五、并发带来的四种事务问题
- 丢失修改
- 脏读
- 不可重复读
- 幻读

#### 二十六、SQL事务隔离级别
- 读未提交
- 读已提交
- 可重复读
- 串行化

#### 二十七、INNODB三类锁
意向锁是为了快速获取当前表是否加锁
1. 记录锁
2. 间隙锁
3. 范围锁

#### 二十八、MySQL 三种日志
binlog（主从同步） 、 redoLog（事务回滚）、undolog（奔溃恢复）

#### 二十九、MyISAM vs InnoDB
1. 行级锁支持
2. 支持事务
3. 崩溃恢复

#### 三十、spring 5种隔离级别、7种传播行为


---
**Redis相关**

#### 一、Redis内存碎片
- 申请内存时会多申请空间
- 清理垃圾时不是非常干净

#### 二、Redis数据结构
- string
- set
- list
- sortset （跳表）
- hash

#### 三、Redis 备份方式
AOF、RDB

#### 四、Redis 清理方式
- 定时删除
- 使用删除

#### 五、Redis 删除过期数据六种策略

#### 六、Redis 使用产生问题即解决方案
问题：缓存穿透、缓存雪崩
方案：1、提高前置拦截检查、2、缓存随机过期

#### 七、缓存一致性问题
结论：先更新数据库然后删除缓存
why：从数据库和缓存执行先后顺序说明


---
**消息队列**

#### 一、作用
- 异步
- 削峰
- 解耦

### 二、副作用
- 降低系统可用性
- 提供系统的复杂性
- 一致性问题

#### 三、常见的消息队列
1. Redis
2. kafka
3. rocketMq
4. rabbitMq

#### 四、RocketMQ 组件
product、broke-master、broke-slave、nameserver、comsumer

#### 五、消息队列常见问题
1. 重复消费
2. 消息堆积
3. 顺序消费
4. 消息丢失

#### 六、事务消息
基于2PC 协议

#### 七、RocketMQ为什么快
- 网络通信：使用netty 
- 储存：fileChannel、buffer、commitLog 1Gb
- 消费：consumerlog 顺序读等


---
**网络**

#### 一、IOS 七层、TCP 四层
- 七层：应用层、表示层、会话层、传输层、网络层、链路层、物理层
- 四层：应用层、传输层、网络层、物理层
- 为什么要分层：
    1. 各层之间相互独立
    2. 提高了整体灵活性
    3. 大问题化小

#### 二、HTTP VS HTTPS

#### 三、HTTP1.0 VS HTTP1.1
- 连接方式：1.0短连接 、 1.1 长连接
- 新增24中响应码
- 缓存处理
- 加入host请求头

#### 四、tcp 三次握手 、四次挥手
- 三次握手：syn、syn/ack、ack
- 四次挥手：fin、ack、 fin 、ack
- 为什么会有三次握手、四次挥手？
面向连接，建立可靠的信道

#### 五、浏览器浏览网页全过程
1. 首先浏览器输入网址，然后host、路由器、DNS缓存中解析目标网址ip
2. 通过ip 建立tcp连接
3. 发送HTTP请求
4. 建立成功后目标服务器响应，最后生成响应报文
5. 浏览器接受报文后渲染页面

#### 六、tcp 如何保证可靠
1. 重试重传
2. ack 机制
3. 阻塞控制
4. 流量控制


---
**操作系统**

#### 一、什么是操作系统？
1.  计算器管理硬件和软件的基石
2.  本质上是运行在计算器上的一套软件系统
3.  屏蔽了硬件层的差异性和复杂性
4.  管理着计算器上的文件系统、内存等

#### 二、系统调用
计算器分为：用户态和系统态，当我们需要系统内存时，需要用户态向系统态发出操作指令，让系统态给我们申请，这一过程称之为系统调用

#### 三、进程和线程
**进程是资源分配的最小单位，线程是CPU调度的最小单位**

#### 四、死锁产生的条件
1. 互斥
2. 等待
3. 非抢占
4. 循环等待

#### 五、操作系统分配内存
- 连续分配：进程预分配一段连续内存
- 非连续分配：逻辑内存和物理内存对应

#### 六、虚拟内存
Linux：swap 空间
- 虚拟内存的重要意义是它定义了一个连续的虚拟地址空间，并且 把内存扩展到硬盘空间


---
**设计模式**
三大类
1. 创建类（5种）
2. 结构类（7种）
3. 行为类（11种）

---
**中间件既框架**
#### 一、spring
1.  prepareRefresh : 加载主类，并设置beandefefiendRead
- **refresh**
2.  obtainFreshBeanFactory
    - 销毁存在的beanFactory
    - 创建新的defaultListBeanFactory
    - 设置是否可以循环依赖、是否可以bean覆盖
    - loadBeanDefinitions 加载所有的bean 定义
3. prepareBeanFactory
    - 设置一些需要特殊处理的bean
    - 初始化一些bean
4. postProcessBeanFactory 注册beanFactoryPostProcess
5. invokeBeanFactoryPostProcessors （实现了自动装配）
6. registerBeanPostProcessors 注册 beanPostProcess
7. finishBeanFactoryInitialization 实例化所有non - lazy bean
    - getBean -> doGetBean
        - getSingleton 三级缓存
    - doCreateBean
        - createBeanInstance
        - populateBean
        - initializeBean
            - applyBeanPostProcessorsBeforeInitialization
            - invokeInitMethods
            - applyBeanPostProcessorsAfterInitialization 在这里实现了aop，利用cglib or jdk 代理实现的
