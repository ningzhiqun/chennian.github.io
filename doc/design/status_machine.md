#### 背景

在需求开发的过程中，经常会遇到根据不同的情况作出不同的处理。最直接的就是if…else…。
当场景特别复杂时，判断if就有些力不从心了。加一个场景需要修改大量的代码，这不是一个很好的做法。程序的扩展性特别薄弱,代码的后期维护也是个大坑。

#### 有限状态机

有限状态机（英语：finite-state machine，缩写：FSM）又称有限状态自动机，简称状态机，是表示有限个状态以及在这些状态之间的转移和动作等行为的数学模型。
站在有限状态机的角度来看，可以抽象如下几个关键点：
状态（State）
当前活动所对应的状态。
事件（Event）
即在触发了某个事件才往下更改状态。
动作（Transition）
即状态流转过程中具体的业务逻辑。

#### 订单状态流转图

![img](/images/design/status_mac_process.png)

订单状态流转图主要表达了事件、动作和状态之间的关系。事件有：提交订单、取消订单、派单、签收等。动作是流转过程中的具体业务逻辑，比如在订单提交时会根据支付类型选择订单可能的状态。上图中绿色方块代表的就是订单状态。

#### 抽象设计

![img](/images/design/level.png)

#### 代码实现

1、事件枚举类：定义触发订单状态流转事件。

```java
package com.pinpianyi.demo.order.state.constant;

import lombok.Getter;

/**
 * 订单事件枚举类
 */
@Getter
public enum OrderEvent {

    SUBMIT(10, "提交订单"),
    CANCEL(11, "取消订单"),
    COMPLETE_PAYMENT(12, "支付完成"),
    CHANGE_ORDER_TYPE(13, "修改订单类型"),
    APPOINTED_ORDER(14, "派单"),
    SIGN_AFTER_RECEIVING(15, "签收");

    private String eventDesc;

    private int eventId;

    OrderEvent(int eventId, String desc) {
        this.eventDesc = desc;
        this.eventId = eventId;
    }

}
```

2、订单状态枚举类：定义订单状态。

```java
package com.pinpianyi.demo.order.state.constant;

public enum OrderStatus {
    /**
     * 未提交
     */
    UN_COMMIT(0, "未提交"),

    /**
     * 等待支付
     */
    WAIT_PAY(100, "待支付"),

    /**
     * 在线支付支付完成
     * 货到付款下单
     */
    WAIT_DISPATCH(200, "待派单/待发货"),

    /**
     * 任一子单已派单，父单置为配送中
     */
    DELIVERING(210, "配送中"),

    /**
     * 全部子单已完成或已关闭，且至少有一单已完成
     */
    COMPLETED(290, "已完成"),

    /**
     * 未知状态
     */
    UN_KNOW(444, "未知"),

    /**
     * 子单全部取消
     */
    CANCELED(295, "已取消");

    OrderStatus(int status, String desc) {
        this.status = status;
        this.desc = desc;
    }

    public int status;

    public String desc;

    public static OrderStatus of(Integer statusCode) {
        for (OrderStatus status : values()) {
            if (status.status == statusCode) {
                return status;
            }
        }
        return null;
    }
}
```

3、支付类型枚举类：定义支付类型。

```java
package com.pinpianyi.demo.order.state.constant;

import lombok.Getter;

@Getter
public enum OrderPayType {

    ONLINE_PAYMENT(0, "在线支付"),

    CASH_ON_DELIVERY(1, "货到付款");

    private int type;

    private String desc;

    OrderPayType(int value, String desc) {
        this.type = value;
        this.desc = desc;
    }
}
```

4、定义注解StatusMigration用于初始化时标识某个类用于处理订单状态流转。

```java
package com.pinpianyi.demo.order.state.core.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 订单状态流传处理器，添加此注解用于配置初始化。
 */
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface StatusMigration {
}
```

5、定义注解AfterProcessor用于初始化时标识某个类用于处理状态流转后的业务逻辑。

```java
package com.pinpianyi.demo.order.state.core.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 订单状态流传操作后处理器标识注解，用于初始化操作
 **/
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface AfterProcessor {
}
```

6、定义抽象类：AbstractOrderStatusMigration、OrderStatusChangedProcessor在初始化操作中完成对应处理器的初始化。

```java
package com.pinpianyi.demo.order.state.core;

import com.pinpianyi.demo.order.state.constant.OrderEvent;
import com.pinpianyi.demo.order.state.constant.OrderPayType;
import lombok.Data;

@Data
public abstract class AbstractOrderStatusMigration {

    private int eventId;

    /**
     * 订单状态流转接口
     *
     * @param orderStatus  当前订单状态
     * @param orderPayType 支付类型
     * @param event        事件
     * @return 流转后的订单状态
     */
    public abstract int handleEvent(int orderStatus, OrderPayType orderPayType, OrderEvent event);
}
```

```java
package com.pinpianyi.demo.order.state.core;

import lombok.Data;

/**
 * 订单状态流转后处理接口
 **/
@Data
public abstract class OrderStatusChangedProcessor {

    private int status;

    public abstract boolean process(String orderId, Object... params);
}
```

7、状态机的核心组件：OrderStatusMigrationManager调用状态流转接口和流转后的业务逻辑接口。

```java
@Component
@Slf4j
public class OrderStatusMigrationManager {

    final Map<Integer, AbstractOrderStatusMigration> orderOperatorMaps = new ConcurrentHashMap<Integer, AbstractOrderStatusMigration>();

    final Map<Integer, OrderStatusChangedProcessor> orderProcessorMaps = new ConcurrentHashMap<Integer, OrderStatusChangedProcessor>();

    public OrderStatusMigrationManager() {
    }

    /**
     * 状态流转方法
     *
     * @param orderId 订单id
     * @param event   流转的订单操作事件
     * @param status  当前订单状态
     * @return 流转后的订单状态
     */
    public int handleEvent(final String orderId, OrderPayType payType, OrderEvent event, final int status) {
        if (this.isFinalStatus(status)) {
            throw new IllegalArgumentException("handle event can't process final state order.");
        }
        log.info(">>>>开始流转orderId={} 的订单状态。<<<<", orderId);
        AbstractOrderStatusMigration abstractOrderStatusMigration = this.getStatusMigrationHandler(event);
        int resState = abstractOrderStatusMigration.handleEvent(status, payType, event);
        log.info(">>>>订单状态流转结束。<<<<");
        OrderStatusChangedProcessor orderProcessor = this.getStatusChangedProcessor(OrderStatus.of(resState));
        log.info(">>>>开始处理状态流转后的业务逻辑。<<<<");
        if (!orderProcessor.process(orderId)) {
            throw new IllegalStateException(String.format("订单状态流转失败，订单id:%s", orderId));
        }
        log.info(">>>>业务处理完成！<<<<<");
        return resState;
    }

    /**
     * 根据入参状态枚举实例获取对应的状态处理器
     *
     * @param event event
     * @return 订单状态处理器
     */
    private AbstractOrderStatusMigration getStatusMigrationHandler(OrderEvent event) {
        AbstractOrderStatusMigration operator = null;
        for (Map.Entry<Integer, AbstractOrderStatusMigration> entry : orderOperatorMaps.entrySet()) {
            if (event.getEventId() == entry.getKey()) {
                operator = entry.getValue();
            }
        }
        if (null == operator) {
            throw new IllegalArgumentException(String.format("can't find proper operator. The parameter state :%s", event.toString()));
        }
        return operator;
    }

    /**
     * 根据入参状态枚举实例获取对应的状态后处理器
     *
     * @param status event
     * @return 状态变更后的业务逻辑处理器
     */
    private OrderStatusChangedProcessor getStatusChangedProcessor(OrderStatus status) {
        OrderStatusChangedProcessor processor = null;
        for (Map.Entry<Integer, OrderStatusChangedProcessor> entry : orderProcessorMaps.entrySet()) {
            if (status.status == entry.getKey()) {
                processor = entry.getValue();
            }
        }
        if (null == processor) {
            throw new IllegalArgumentException(String.format("can't find proper processor. The parameter state :%s", status.toString()));
        }
        return processor;
    }


    /**
     * 判断订单是否可流转
     *
     * @param status 订单状态码
     * @return
     */
    private boolean isFinalStatus(int status) {
        return OrderStatus.COMPLETED.status == status || OrderStatus.UN_KNOW.status == status || OrderStatus.CANCELED.status == status;
    }

}
```

8、实现BeanPostProcessor接口，将具体的状态流转实现类和流转后的处理器注册到OrderStatusMigrationManager中。

```java
@Component
public class Initialization implements BeanPostProcessor {

    @Resource
    OrderStatusMigrationManager manager;

    @Nullable
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof AbstractOrderStatusMigration && bean.getClass().isAnnotationPresent(StatusMigration.class)) {
            AbstractOrderStatusMigration orderOperator = (AbstractOrderStatusMigration) bean;
            manager.orderOperatorMaps.put(orderOperator.getEventId(), orderOperator);
        }
        if (bean instanceof OrderStatusChangedProcessor && bean.getClass().isAnnotationPresent(AfterProcessor.class)) {
            OrderStatusChangedProcessor orderProcessor = (OrderStatusChangedProcessor) bean;
            manager.orderProcessorMaps.put(orderProcessor.getStatus(), orderProcessor);
        }
        return bean;
    }
}
```

至此，基于状态机的订单状态流转的核心逻辑已介绍完毕，下面按照之前的设计实现一个具体的创建订单业务用于测试。

#### 测试验证

1、定义Order

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Order {

    private String orderId;

    private int orderStatus;
}
```

2、实现订单状态流转接口

```java
@Component
@StatusMigration
@Slf4j
public class CreateOrderStatusMigration extends AbstractOrderStatusMigration {

    public CreateOrderStatusMigration() {
        super.setEventId(OrderEvent.SUBMIT.getEventId());
    }

    @Override
    public int handleEvent(int orderStatus, OrderPayType payType, OrderEvent event) {
        log.info("订单当前状态是：{}，订单支付类型是：{}，触发订单状态流转事件是：{}", orderStatus, payType.getDesc(), event.getEventDesc());
        if (orderStatus != OrderStatus.UN_COMMIT.status || event != OrderEvent.SUBMIT) {
            throw new IllegalArgumentException(String.format("create operation can't handle the status: %s", orderStatus));
        }
        OrderStatus status;
        switch (payType) {
            case ONLINE_PAYMENT:
                status = OrderStatus.WAIT_PAY;
                break;
            case CASH_ON_DELIVERY:
                status = OrderStatus.WAIT_DISPATCH;
                break;
            default:
                status = OrderStatus.UN_KNOW;
        }
        log.info("流转后的订单状态是：{}，状态码:{}", status.desc, status.status);
        return status.status;
    }
}
```

3、实现订单状态流转后的业务逻辑处理接口

```java
@Component
@AfterProcessor
@Slf4j
public class CreateOrderStatusChangedProcessor extends OrderStatusChangedProcessor {

    public CreateOrderStatusChangedProcessor() {
        setStatus(OrderStatus.WAIT_PAY.status);
    }

    @Override
    public boolean process(String orderId, Object... params) {
        log.info("订单号：{} 的订单进入创建后处理器...", orderId);
        log.info("说明：创建/取消订单对应的数据库修改，mq发送等操作，可以在此处process方法中完成");
        return true;
    }
}
```

4、增加一个SpringBeanConfig用于bean实列化

```java
@ComponentScan
@Configuration
public class SpringBeanConfig {

    @Bean
    public Initialization initialization() {
        return new Initialization();
    }

    @Bean
    public OrderStatusMigrationManager orderStatusMigrationManager() {
        return new OrderStatusMigrationManager();
    }
}
```

5、编写单元测试用列代码

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = SpringBeanConfig.class)
@Slf4j
public class StateMachineTest {

    @Autowired
    private OrderStatusMigrationManager orderStatusMigrationManager;

    private Order order;

    @Before
    public void init() {
        order = new Order("20201214000001", OrderStatus.UN_COMMIT.status);
    }

    @Test
    public void createOrder() {
        Assert.assertEquals(100, orderStatusMigrationManager.handleEvent(order.getOrderId(), OrderPayType.ONLINE_PAYMENT, OrderEvent.SUBMIT, order.getOrderStatus()));
    }
}
```

运行createOrder方法可得到如下结果：

```rst
1148 [main] DEBUG org.springframework.test.context.support.AbstractDirtiesContextTestExecutionListener  - Before test method: context [DefaultTestContext@25bbe1b6 testClass = StateMachineTest, testInstance = com.pinpianyi.demo.order.test.StateMachineTest@2d928643, testMethod = createOrder@StateMachineTest, testException = [null], mergedContextConfiguration = [MergedContextConfiguration@5702b3b1 testClass = StateMachineTest, locations = '{}', classes = '{class com.pinpianyi.demo.order.SpringBeanConfig}', contextInitializerClasses = '[]', activeProfiles = '{}', propertySourceLocations = '{}', propertySourceProperties = '{}', contextCustomizers = set[[empty]], contextLoader = 'org.springframework.test.context.support.DelegatingSmartContextLoader', parent = [null]], attributes = map[[empty]]], class annotated with @DirtiesContext [false] with mode [null], method annotated with @DirtiesContext [false] with mode [null].
1151 [main] INFO  com.pinpianyi.demo.order.state.core.OrderStatusMigrationManager  - >>>>开始流转orderId=20201214000001 的订单状态。<<<<
1151 [main] INFO  com.pinpianyi.demo.order.state.migration.impl.CreateOrderStatusMigration  - 订单当前状态是：0，订单支付类型是：在线支付，触发订单状态流转事件是：提交订单
1153 [main] INFO  com.pinpianyi.demo.order.state.migration.impl.CreateOrderStatusMigration  - 流转后的订单状态是：待支付，状态码:100
1153 [main] INFO  com.pinpianyi.demo.order.state.core.OrderStatusMigrationManager  - >>>>订单状态流转结束。<<<<
1154 [main] INFO  com.pinpianyi.demo.order.state.core.OrderStatusMigrationManager  - >>>>开始处理状态流转后的业务逻辑。<<<<
1154 [main] INFO  com.pinpianyi.demo.order.state.processor.impl.CreateOrderStatusChangedProcessor  - 订单号：20201214000001 的订单进入创建后处理器...
1154 [main] INFO  com.pinpianyi.demo.order.state.processor.impl.CreateOrderStatusChangedProcessor  - 说明：创建/取消订单对应的数据库修改，mq发送等操作，可以在此处process方法中完成
1154 [main] INFO  com.pinpianyi.demo.order.state.core.OrderStatusMigrationManager  - >>>>业务处理完成！<<<<<
1157 [main] DEBUG org.springframework.test.context.support.AbstractDirtiesContextTestExecutionListener  - After test method: context [DefaultTestContext@25bbe1b6 testClass = StateMachineTest, testInstance = com.pinpianyi.demo.order.test.StateMachineTest@2d928643, testMethod = createOrder@StateMachineTest, testException = [null], mergedContextConfiguration = [MergedContextConfiguration@5702b3b1 testClass = StateMachineTest, locations = '{}', classes = '{class com.pinpianyi.demo.order.SpringBeanConfig}', contextInitializerClasses = '[]', activeProfiles = '{}', propertySourceLocations = '{}', propertySourceProperties = '{}', contextCustomizers = set[[empty]], contextLoader = 'org.springframework.test.context.support.DelegatingSmartContextLoader', parent = [null]], attributes = map[[empty]]], class annotated with @DirtiesContext [false] with mode [null], method annotated with @DirtiesContext [false] with mode [null].
```

测试结果符合预期，如果要测试更多的状态流转，只需要实现相关状态流转接口和流转后的业务逻辑处理类就可以了。这种设计方式非常适合处理有限状态的流转，可以将状态流转、流转后的处理逻辑与其它业务逻辑分离。
