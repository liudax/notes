# Spring事务管理

## 事务概念回顾
>
>  **什么是事务？**
>

事务是逻辑上的一组操作，要么都执行，要么都不执行。



>**事务的特性(ACID)：**

![](images\ACID.png)

**原子性(Atomicity)**：事务是最小执行单位，不允许分割。事务的原子性确保动作要么全部完成，要么全部不执行。

**一致性(Consistency)**：执行事务前后，数据保持一致；事务必须始终保持系统处于一致的状态，不管在任何给定的时间并发事务有多少。

**隔离性(Isolation)**：并发访问数据库时，一个用户的事务不会被其他事务所干扰，各并发事务之间是相互独立的。

**持久性(Durability)**：一个事务被提交后，那么它对数据库中的改变是持久的，即使数据库中发生故障也不应该对其有任何影响。



## Spring事务管理接口

 ### 

* **PlatformTransactionManager**:（平台）事务管理器
* **TransactionDefinition**：事务定义信息（事务隔离级别、传播行为、超时、只读、回滚原则）
* **TransactionStatus**:事务运行状态

**所谓事务管理，其实就是“按照给定的事务规则来执行提交或者回滚操作**



 ### PlatformTransactionManager

**Spring并不直接管理事务，而是提供了多种事务管理器**。它将事务管理的职责委托给了hibernate或者JTA等持久化机制 所提供的相关平台框架的事务来实现。Spring事务管理器的接口是：**org.springframework.transaction.PlatformTransactionManager** ，通过这个接口，Spring为各个平台如JDBC、Hibernate等都提供了对应的事务管理器，但是具体的实现就是各个平台自己的事情了。

PlatformTransactionManager接口中定义了三个方法：

```java
Public interface PlatformTransactionManager()...{  
    // Return a currently active transaction or create a new one, according to the specified propagation behavior（根据指定的传播行为，返回当前活动的事务或创建一个新事务。）
    TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException; 
    // Commit the given transaction, with regard to its status（使用事务目前的状态提交事务）
    Void commit(TransactionStatus status) throws TransactionException;  
    // Perform a rollback of the given transaction（对执行的事务进行回滚）
    Void rollback(TransactionStatus status) throws TransactionException;  
    } 

```



不同持久层框架事务管理的实现，常见的如下：

![](images\事务管理实现.png)

事务配置通常如下：

```xml
<!-- 事务管理器 -->
	<bean id="transactionManager"
		class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		<!-- 数据源 -->
		<property name="dataSource" ref="dataSource" />
	</bean>

```

或者注解Configuration

```java
 	@Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource){
        DataSourceTransactionManager transactionManager = new 			      DataSourceTransactionManager();
        transactionManager.setDataSource(dataSource);
        return transactionManager;
    }
```

PlatformTransactionManager通过**getTransaction(TransactionDefinition definition)**方法获取一个事务，TransactionDefinition 这个类定义了一些基本的事务属性。

### TransactionDefinition 

> **什么是事务属性呢**

事务属性可以理解成事务的一些基本配置，描述了事务策略如何应用到方法上。事务属性包含了5个方面。

![](images\事务属性1.png)





TransactionDefinition 定义了5个方法以及一些表示事务属性的常量比如隔离级别，传播行为等。

方法：

```java
public interface TransactionDefinition {
    // 返回事务的传播行为
    int getPropagationBehavior(); 
    // 返回事务的隔离级别，事务管理器根据它来控制另外一个事务可以看到本事务内的哪些数据
    int getIsolationLevel(); 
    // 返回事务必须在多少秒内完成
    //返回事务的名字
    String getName()；
    int getTimeout();  
    // 返回是否优化为只读事务。
    boolean isReadOnly();
} 
```



> **并发引起的问题**

介绍之前，先看一下**并发事务带来的问题**，然后在介绍五个隔离级别的常量。

在典型的应用程序中，多个事务并发运行，经常会操作相同的数据来完成各自的操作（多个用户对同一数据进行操作）。并发是必须的，但可能会导致以下问题：

* **脏读（Dirty read）**:当一个事务A正在访问数据并对这个数据进行了修改，而没有提交。此时另外一个事务B也访问了这个数据并使用。因为这个修改过后的数据是没有提交的，那么B事务读到的就是”脏数据“，依据”脏数据“所做的操作可能是不正确的。

* **丢失修改（Lost to modify)**：指事务A读取一个数据时，事务B也读取了该数据。那么A事务修改了该数据，B事务也修改了该数据，这样的话，事务A修改的结果就丢失了，因此称谓丢失修改。

  例如：事务1读取某表中的数据A=20，事务2也读取A=20，事务1修改A=A-1，事务2也修改A=A-1，最终结果A=19，事务1的修改被丢失。

* **不可重复读（Unrepeatableread）**：当事务A多次读同一条数据 。在这个事务没有结束时，事务B也访问该数据并修改。那么在事务A两次读取数据的期间，由于事务B修改了数据，会导致事务A 两次读取的数据可能不一样。这样就导致了同一个事务内两次读到的数据可能不一样，因此称为不可重复读。

* **幻读（Phantom read）**：幻读与不可重复读类似。它发生在一个事务A读取了几行数据，接着另一个并发事务B插入了一些数据时。在随后的查询中，第一个事务A就会发现多了一些原本不存在的记录，就好像发生了幻觉一样，所以称为幻读。

不可重复读和幻读的主要区别在与：**不可重复读重点在于修改，幻读重点在于新增和删除**。



#### 隔离级别

| 事务隔离级别 | 回滚覆盖 | 脏读     | 不可重复读 | 提交覆盖 | 幻读     |
| ------------ | -------- | -------- | ---------- | -------- | -------- |
| 读未提交     | x        | 可能发生 | 可能发生   | 可能发生 | 可能发生 |
| 读已提交     | x        | x        | 可能发生   | 可能发生 | 可能发生 |
| 可重复读     | x        | x        | x          | x        | 可能发生 |
| 串行化       | x        | x        | x          | x        | x        |

TransactionDefinition中定义了五个表示隔离级别的常量：

* **TransactionDefinition.ISOLATION_DEFAULT**:使用后端**数据库默认的隔离级别**，Mysql采用的可重复读（REPEATABLE_READ）。 Oracle 默认采用的 读已提交（READ_COMMITTED）。
* **TransactionDefinition.ISOLATION_READ_UNCOMMITED**：最低的隔离级别，允许读取未提交的数据，可能会**造成脏读，幻读或者不可重复读。**
* **TransactionDefinition.ISOLATION_READ_COMMITTED:** 允许读取并发事务已经提交的数据，**可以阻止脏读，但是幻读或不可重复读仍有可能发生**
* **TransactionDefinition.ISOLATION_REPEATABLE_READ:** 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，**可以阻止脏读和不可重复读，但幻读仍有可能发生。**
* **TransactionDefinition.ISOLATION_SERIALIZABLE:** 	最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，**该级别可以防止脏读、不可重复读以及幻读**。但是这将严重影响程序的性能。通常情况下也不会用到该级别。



#### 传播行为

*为了解决业务层方法之间互相调用的事务问题*

当事务方法被另一个事务方法调用时，必须指定事务应该如何传播。例如：方法可能继续在现有事务中运行，也可能开启一个新事务，并在自己的事务中运行。在TransactionDefinition定义中包括了如下几个表示传播行为的常量：

**支持当前事务的情况：**

* TransactiondDefinition.PROPAGATION_REQUIRED：如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新事务。
* TransactionDefinition.PROPAGATION_SUPPORTS：如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式运行。
* TransactionDefinition.PROPAGATION_MANDATORY：如果当前存在事务，则加入该事务；如果当前没有事务，抛出异常。

**不支持当前事务的情况**

* TransactionDefinition.PROPAGATION_REQUIRES_NEW：创建一个新事务，如果当前存在事务，则把当前事务挂起。
* TransactionDefinition.PROPAGATION_NOT_SUPPORTED：以非事务方式运行，如果当前存在事务，则把当前事务挂起。
* TransactionDefinition.PROPAGATION_NEVER：以非事务方式运行，如果当前存在事务，抛出异常。

**其他情况**

* TransactionDefinition.PROPAGATION_NESTED： 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。

这里需要指出的是，前面的六种事务传播行为是 Spring 从 EJB 中引入的，他们共享相同的概念。而 **PROPAGATION_NESTED** 是 Spring 所特有的。以 PROPAGATION_NESTED 启动的事务内嵌于外部事务中（如果存在外部事务的话），此时，内嵌事务并不是一个独立的事务，它依赖于外部事务的存在，只有通过外部的事务提交，才能引起内部事务的提交，嵌套的子事务不能单独提交。如果熟悉 JDBC 中的保存点（SavePoint）的概念，那嵌套事务就很容易理解了，其实嵌套的子事务就是保存点的一个应用，一个事务中可以包括多个保存点，每一个嵌套子事务。另外，外部事务的回滚也会导致嵌套子事务的回滚。



#### 事务超时属性

指一个事务允许执行的最长时间，如果超过该时间限制事务还没有完成，则自动回滚事务，TransactionDefinition中以int来表示超时时间，其单位是秒。



#### 事务只读属性

指对事务只读操作或者读些操作。所谓事务性资源就是指那些被事务管理的资源，比如数据源、JMS资源以及自定义的事务资源等等。如果确定只对事务资源进行读操作，那么将事务标记为只读的，以提高事务处理性能。在TransactionDefinition 中以boolean类型表示是否只读。



#### 回滚规则

这些规则定义了哪些异常会导致事务回滚，哪些不会。默认情况下，只有遇到运行时期异常才会回滚，检查性不会。但可以声明遇到哪些异常才会回滚，遇到哪些异常不会滚。这些异常包括运行时期和检查性异常。



### TransactionStatus接口介绍

TransactionStatus接口用来记录事务的状态，该接口定义了一组方法，用来获取或判断事务相应的信息。方法如下
```java
public interface TransactionStatus{
    boolean isNewTransaction(); // 是否是新的事物
    boolean hasSavepoint(); // 是否有恢复点
    void setRollbackOnly();  // 设置为只回滚
    boolean isRollbackOnly(); // 是否为只回滚
    boolean isCompleted; // 是否已完成
} 
```

PlatformTransactionManager.getTransaction(...) 返回一个TransactionStatus对象，是当前堆栈（当前线程）新的或者已经存在的事务的状态信息。








原文地址 [https://juejin.im/post/5b00c52ef265da0b95276091](https://juejin.im/post/5b00c52ef265da0b95276091)