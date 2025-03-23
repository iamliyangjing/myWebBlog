---
description: Spring事务相关内容
title: Spring 事务
tag:
  - Spring
  - MySQL
sidebar: true
comment: true
recommend: 1
---
Spring 事务

### 1. 事务的基本概念

#### 事务四个基础特性-ACID

原子性（Atomicity）：原子性要求事务被视为不可分割的最小工作单元，要么全部执行成功，要么全部失败回滚。事务在执行过程中发生错误或中断，系统必须能够将其恢复到事务开始前的状态，保证数据的一致性。

一致性（Consistency）：一致性确保事务在执行前后数据库的状态是一致的。事务在执行过程中对数据库进行的修改必须满足预定义的规则和约束，以保证数据的完整性。

隔离性（Isolation）：隔离性指多个事务并发执行时，每个事务的操作都应当与其他事务相互隔离，使它们感觉不到其他事务的存在。隔离性可以防止并发执行的事务之间发生干扰和数据冲突，确保数据的正确性。

持久性（Durability）：持久性要求事务一旦提交，其对数据库的修改就是永久性的，即使在系统发生故障或重启的情况下，修改的数据也能够被恢复。持久性通过将事务的结果写入非易失性存储介质（如磁盘）来实现。

#### 四种隔离级别

- **READ_UNCOMMITTED**：最低隔离级别，允许读取未提交的数据变更。在该级别下，一个事务可以读取到另一个事务未提交的数据，可能导致**脏读**，即读取到了未经验证的数据。这个级别会导致数据的不一致性，并且不提供任何并发控制。
- **READ_COMMITTED**：只能读取已提交的数据变更。它避免了脏读，但可能出现不可重复读（Non-repeatable Read）的问题。**不可重复读**是指同一个事务中多次读取同一数据，在事务执行过程中，该数据被其他事务修改，导致每次读取到的值不一致。
- **REPEATABLE_READ**：确保同一事务中多次读取同一数据结果一致。
- 使其他事务对该数据进行修改，也不会影响当前事务的读取操作。这个级别通过锁定读取的数据，避免了不可重复读，但可能出现幻读（Phantom Read）的问题。**幻读**是指同一个事务中多次查询同一个范围的数据时，由于其他事务插入了新的数据，导致每次查询结果集不一致。
- **SERIALIZABLE**：最高隔离级别，确保事务串行执行,完全避免了并发问题。在该级别下，事务之间互相看不到对方的操作，可以避免脏读、不可重复读和幻读等问题。然而，由于串行化执行，会牺牲一定的性能。

### 2. Spring事务

#### Spring如何开启事务

spring开启事务的方式有两种：

1. 声明式事务(**使用注解**)
2. 编程式事务

声明式事务是如何实现的：

1. Spring事务底层是基于数据库事务和AOP机制的
2. 首先对于使用了@Transactional注解的Bean，Spring会创建一个代理对象作为Bean
3. 当调用代理对象的方法时，会先判断该方法上是否有@Transactional
4. 如果加了，那么**则利用事务管理器创建一个数据库连接**
5. **并且修改数据库连接的autocommit的属性为false，禁止此连接的自动提交，这是实现spring事务非常重要的一步**
6. 然后执行当前方法，方法中会执行SQL
7. 执行完当前方法后，如果没有出现异常就直接提交事务
8. 如果出现了异常，并且这个异常是需要回滚的就会回滚事务，否则仍然提交事务
9. Spring事务的隔离级别对应的就是数据库的隔离级别
10. Spring事务的传播机制是Spring事务自己实现的，也是Spring事务中最复杂的
11. Spring事务的传播机制是基于数据库连接来做的，一个数据库连接一个事务，如果传播机制配置为需要新开一个事务，那么实际上就是先建立一个新的数据库连接，在上面执行SQL。

#### Transactional 注解

- **使用场景**：在方法或类上使用`@Transactional`注解，Spring会自动管理事务。
- **属性配置**：
  - `propagation`：事务传播行为。
  - `isolation`：事务隔离级别。
  - `readOnly`：是否只读事务。
  - `timeout`：事务超时时间。
  - `rollbackFor`：指定哪些异常触发回滚。
  - `noRollbackFor`：指定哪些异常不触发回滚。

#### Spring事务的传播行为

- **REQUIRED**：如果当前存在事务，则加入该事务；否则创建一个新事务。
- **REQUIRES_NEW**：总是创建一个新事务，如果当前存在事务，则挂起当前事务。
- **NESTED**：如果当前存在事务，则在嵌套事务中执行；否则创建一个新事务。
- **SUPPORTS**：如果当前存在事务，则加入该事务；否则以非事务方式执行。
- **NOT_SUPPORTED**：以非事务方式执行，如果当前存在事务，则挂起当前事务。
- **MANDATORY**：如果当前存在事务，则加入该事务；否则抛出异常。
- **NEVER**：以非事务方式执行，如果当前存在事务，则抛出异常。

![image-20250323225828431](D:\Git\myWebSite\myWebSite\docs\框架\Spring\Spring事务.assets\image-20250323225828431.png)

#### 嵌套事务和加入事务的区别

1. 嵌套事务：嵌套事务是指一个事务内部开启了一个独立的事务，这个事务出现异常不会引起外部的回滚，并且具有独立的事务日志和回滚机制。 嵌套事务允许在父事务中进行更细粒度的操作和控制，例如，在一个长事务中的某个步骤中开启了一个子事务，子事务可以独立提交或回滚，而不会影响父事务的其他步骤。嵌套事务通常用于复杂的业务逻辑，可以提供更灵活的事务处理。
2. 加入事务：加入事务是指将一个独立的事务加入到当前的事务中，使他们成为一个整体。加入事务可以将多个事务合并为一个更大的事务，确保它们作为一个原子操作进行提交或回滚。加入事务通常用于多个独立事务之间存在逻辑上的依赖关系，需要以一致的方式进行处理。通过将多个事务加入到一个事务中，可以保证它们的一致性，**并且要么全部提交成功，要么全部回滚。**

### 3. Spring事务失效的场景

1. 方法内的自调用

2. 1. **把调用方法拆分到另外一个Bean中**
   2. **自己注入自己。**
   3. **AopContext.currentProxy + @EnableAspectAutoProxy(exposeProxy = true)。**

3. 方法是private的：Spring事务会基于CGLIB来进行AOP，而CGLIB会基于父子类来失效，子类是代理类，父类是被代理类，如果父类中的某个方法是private的，那么子类就没有办法重写
4. 方法是final的.
5. 单独的线程调用方法。
6. 没加@Configuration注解。
7. 异常被吃掉。
8. 类没有被Spring管理。
9. 数据库不支持事务。

#### 事务方法未被Spring管理

如果事务方法所在的类没有注册到`Spring IOC`容器中，也就是说，事务方法所在类并没有被`Spring`管理，则`Spring`事务会失效，举个例子🌰：

```java
public class ProductServiceImpl extends ServiceImpl<ProductMapper, Product> implements IProductService {

    @Autowired
    private ProductMapper productMapper;

    @Override
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updateProductStockById(Integer stockCount, Long productId) {
        productMapper.updateProductStockById(stockCount, productId);
    }
}
```

`ProductServiceImpl`实现类上没有添加`@Service`注解，`Product`的实例也就没有被加载到`Spring IOC`容器，此时`updateProductStockById()`方法的事务就会在`Spring`中失效。

#### 方法使用final类型修饰

有时候，某个方法不想被子类重新，这时可以将该方法定义成`final`的。普通方法这样定义是没问题的，但如果将事务方法定义成`final`，例如：

```java
@Service
public class OrderServiceImpl {

    @Transactional
    public final void cancel(OrderDTO orderDTO) {
        // 取消订单
        cancelOrder(orderDTO);
    }
}
```

`OrderServiceImpl`的`cancel`取消订单方法被`final`修饰符修饰，`Spring`事务底层使用了`AOP`，也就是通过`JDK`动态代理或者`cglib`，帮我们生成了代理类，在代理类中实现的事务功能。但如果某个方法用`final`修饰了，那么在它的代理类中，就无法重写该方法，从而无法添加事务功能。这种情况事务就会在`Spring`中失效。

> 💡**Tips: 如果某个方法是`static`的，同样无法通过动态代理将方法声明为事务方法。**

#### 非Public 修饰的方法

如果事务方式不是`public`修饰，此时`Spring`事务会失效，举个例子🌰：

```java
@Service
public class ProductServiceImpl extends ServiceImpl<ProductMapper, Product> implements IProductService {

    @Autowired
    private ProductMapper productMapper;

    @Override
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    private void updateProductStockById(Integer stockCount, String productId) {
        productMapper.updateProductStockById(stockCount, productId);
    }
}

```

虽然`ProductServiceImpl`添加了`@Service`注解，同时`updateProductStockById()`方法上添加了`@Transactional(propagation = Propagation.REQUIRES_NEW)`注解，但是由于事务方法`updateProductStockById()`被 **`private`** 定义为方法内私有，同样`Spring`事务会失效。

#### 同一个类中方法互相调用

```java
@Service
public class OrderServiceImpl extends ServiceImpl<OrderMapper, Order> implements IOrderService {
    @Autowired
    private OrderMapper orderMapper;
    @Autowired
    private ProductMapper productMapper;

    @Override
    public ResponseEntity submitOrder(Order order) {
        // 保存生成订单信息
        long orderNo = Math.abs(ThreadLocalRandom.current().nextLong(1000));
        order.setOrderNo("ORDER_" + orderNo);
        orderMapper.insert(order);

        // 扣减库存
        this.updateProductStockById(order.getProductId(), 1L);
        return new ResponseEntity(HttpStatus.OK);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updateProductStockById(Integer num, Long productId) {
        productMapper.updateProductStockById(num, productId);
    }
}

```

`submitOrder()`方法和`updateProductStockById()`方法都在`OrderService`类中，然而`submitOrder()`方法没有添加事务注解，`updateProductStockById()`方法虽然添加了事务注解，这种情况`updateProductStockById()`会在`Spring`事务中失效。



#### 

#### 数据库存储引擎不支持事务

顾名思义，有一些数据库存储引擎就不支持事务，比如InnoDB。

#### 异常被内部catch，程序生吞异常

```java
@Service
public class OrderServiceImpl extends ServiceImpl<OrderMapper, Order> implements IOrderService {
    @Autowired
    private OrderMapper orderMapper;
    @Autowired
    private ProductMapper productMapper;

    @Override
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public ResponseEntity submitOrder(Order order) {
        long orderNo = Math.abs(ThreadLocalRandom.current().nextLong(1000));
        order.setOrderNo("ORDER_" + orderNo);
        orderMapper.insert(order);

        // 扣减库存
        this.updateProductStockById(order.getProductId(), 1L);
        return new ResponseEntity(HttpStatus.OK);
    }

    /**
     * 扣减库存方法事务类型声明为NOT_SUPPORTED不支持事务的传播
     */
    @Transactional(propagation = Propagation.NOT_SUPPORTED)
    public void updateProductStockById(Integer num, Long productId) {
        try {
            productMapper.updateProductStockById(num, productId);
        } catch (Exception e) {
            // 这里仅仅是捕获异常之后的打印（相当于程序吞掉了异常）
            log.error("Error updating product Stock: {}", e);
        }
    }

```

#### 配置未开启事务

如果项目中没有配置`Spring`的事务管理器，即使使用了`Spring`的事务管理功能，`Spring`的事务也不会生效，例如，如果你是`Spring Boot`项目，没有在`SpringBoot`项目中配置如下代码：

```java
@Bean
public PlatformTransactionManager transactionManager(DataSource dataSource) {
    return new DataSourceTransactionManager(dataSource);
}
```

如果是以往的`Spring MVC`项目，如果没有配置下面的代码，`Spring`事务也不会生效，正常需要在`applicationContext.xml`文件中，手动配置事务相关参数，比如：

```java
<!-- 配置事务管理器 --> 
<bean class="org.springframework.jdbc.datasource.DataSourceTransactionManager" id="transactionManager"> 
    <property name="dataSource" ref="dataSource"></property> 
</bean> 
<tx:advice id="advice" transaction-manager="transactionManager"> 
    <tx:attributes> 
        <tx:method name="*" propagation="REQUIRED"/>
    </tx:attributes> 
</tx:advice> 
<!-- 用切点把事务切进去 --> 
<aop:config> 
    <aop:pointcut expression="execution(* com.universal.ubdk.*.*(..))" id="pointcut"/> 
    <aop:advisor advice-ref="advice" pointcut-ref="pointcut"/> 
</aop:config> 
```

#### 错误的传播特性

其实，我们在使用`@Transactional`注解时，是可以指定`propagation`参数的。

该参数的作用是指定事务的传播特性，目前`Spring`支持7种传播特性：

- `REQUIRED` 如果当前上下文中存在事务，那么加入该事务，如果不存在事务，创建一个事务，这是默认的传播属性值。
- `SUPPORTS` 如果当前上下文存在事务，则支持事务加入事务，如果不存在事务，则使用非事务的方式执行。
- `MANDATORY` 如果当前上下文中存在事务，否则抛出异常。
- `REQUIRES_NEW` 每次都会新建一个事务，并且同时将上下文中的事务挂起，执行当前新建事务完成以后，上下文事务恢复再执行。
- `NOT_SUPPORTED` 如果当前上下文中存在事务，则挂起当前事务，然后新的方法在没有事务的环境中执行。
- `NEVER` 如果当前上下文中存在事务，则抛出异常，否则在无事务环境上执行代码。
- `NESTED` 如果当前上下文中存在事务，则嵌套事务执行，如果不存在事务，则新建事务。

如果我们在手动设置`propagation`参数的时候，把传播特性设置错了，比如：

```java
@Service
public class OrderServiceImpl {

    @Transactional(propagation = Propagation.NEVER)
    public void cancelOrder(UserModel userModel) {
        // 取消订单
        cancelOrder(orderDTO);
        // 还原库存
        restoreProductStock(orderDTO.getProductId(), orderDTO.getProductCount());
    }
}
```

我们可以看到`cancelOrder()`方法的事务传播特性定义成了`Propagation.NEVER`，这种类型的传播特性不支持事务，如果有事务则会抛异常

#### 多线程调用

在实际项目开发中，多线程的使用场景还是挺多的。如果`Spring`事务用在多线程场景中使用不当，也会导致事务无法生效。

```java
@Slf4j
@Service
public class OrderServiceImpl {

    @Autowired
    private OrderMapper orderMapper;
    @Autowired
    private MessageService messageService;

    @Transactional
    public void orderCommit(orderModel orderModel) throws Exception {
        orderMapper.insertOrder(orderModel);
        new Thread(() -> {
            messageService.sendSms();
        }).start();
    }
}

@Service
public class MessageService {

    @Transactional
    public void sendSms() {
        // 发送短信
    }
}
```

通过示例，我们可以看到订单提交的事务方法`orderCommit()`中，调用了发送短信的事务方法`sendSms()`，但是发送短信的事务方法`sendSms()`是另起了一个线程调用的。

这样会导致两个方法不在同一个线程中，从而是两个不同的事务。如果是`sendSms()`方法中抛了异常，`orderCommit()`方法也回滚是不可能的。

实际上，`Spring`的事务是通过`ThreadLocal`来保证线程安全的，事务和当前线程绑定，多个线程自然会让事务失效。