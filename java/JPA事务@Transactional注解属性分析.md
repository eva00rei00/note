# JPA事务@Transactional注解属性分析

本文介绍JPA事务注解方式使用的各项属性配置作用和适用场景.主要分析了传播传播行为和隔离级别的具体作用和使用情况.

[TOC]

## 1. 事务开启方式
1. **xml配置方式启动JPA事务事务管理器**
``` xml
  <tx:annotation-driven/>
  <bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager">
    <property name="entityManagerFactory" ref="entityManagerFactory"/>
  </bean>
```
2. **在需要使用事务控制的具体方法上使用@Transactional 注解,并根据实际情况配置各项属性.**
## 2. @Transactional 注解的属性

| 属性名     |  说明          |
| :-------- | :------------- |
| value或transactionManager       | 当在配置文件中有多个 TransactionManager , 可以用该属性指定选择哪个事务管理器。  |
| propagation      | 事务的传播行为，默认值为 PROPAGATION_REQUIRED。 |
| isolation      |  事务的隔离级别，默认值采用 DEFAULT。使用底层数据库默认隔离级别        |
| timeout      |  事务的超时时间，默认值为-1,表示超时将依赖于底层事务系统。如果超过该时间限制但事务还没有完成，则自动回滚事务。        |
| read-only      |  指定事务是否为只读事务，默认为false.只有查询语句的方法开启此设置可优化数据库开销,并防止不可重复读.        |
| rollback-for      |  用于指定能够触发事务回滚的异常类型，如果有多个异常类型需要指定，各类型之间可以通过逗号分隔。默认为发生RuntimeException时回滚事务.        |
| no-rollback-for      |  抛出 no-rollback-for 指定的异常类型，不回滚事务。        |

### 2.1 propagation事务传播行为

> 此属性的作用是用来决定当其他方法调用事务方法时新的事务方法是否加入原有事务环境,或者创建事务新的事务环境,或者运行在非事务环境.
> TransactionDefinition中定义了7种传播行为,默认值为PROPAGATION_REQUIRED。

``` java
int PROPAGATION_REQUIRED = 0;
int PROPAGATION_SUPPORTS = 1;
int PROPAGATION_MANDATORY = 2;
int PROPAGATION_REQUIRES_NEW = 3;
int PROPAGATION_NOT_SUPPORTED = 4;
int PROPAGATION_NEVER = 5;
int PROPAGATION_NESTED = 6;
```

#### 2.1.1 各传播行为的具体涵义

假设子方法有@Transactional注解,有其他方法内部调用子方法,那么根据父方法是否存在事务和子方法不同传播行为配置会有以下几种情况:

| 子方法配置的传播行为     |  父方法存在事务          | 父方法不存在事务          |
| :-------- | :------------- | :------------- |
| REQUIRED       | 加入当前事务执行  | 新建一个事务执行 |
| REQUIRES_NEW       | 将当前事务挂起,并创建了一个新事务执行  | 新建一个事务执行 |
| NESTED       | 在当前事务中嵌套一个子事务执行  | 新建一个事务执行 |
| MANDATORY       | 加入当前事务执行  | 抛出异常 |
| SUPPORTS       | 加入当前事务执行  | 以非事务方式执行 |
| NOT_SUPPORTED       | 将当前事务挂起,并以非事务方式执行  | 以非事务方式执行 |
| NEVER       | 抛出异常  | 以非事务方式执行 |

#### 2.1.2 各传播行为的具体表现和使用场景

##### 2.1.2.1 REQUIRED

>表示该方法会加入原有的事务环境,如果没有事务环境就会在一个新的事务环境中执行.

| 事务情况 | 子方法抛出异常父方法不捕获 | 子方法抛出异常父方法捕获并处理 | 父方法抛出异常 |
| :-------- | :-------- | :-------- | :-------- |
|父方法有事务 -> 子方法加入事务|事务回滚|事务回滚(rollback only)|事务回滚|
|父方法无事务 -> 子方法新建事务|子方法事务回滚|子方法事务回滚|无事务回滚|
* 父方法有事务的情况下,当子方法抛出异常后,整个事务就会被标记为rollback only,无论是否处理异常,整个事务都只能被回滚,如果提交会抛出运行时异常TransactionException.

###### 使用方法
默认的,同时也是绝大多数情况适用的传播行为,可以简单地控制各个事务方法间的调用会被控制在同一个事务中.
绝大多数service层的public对外接口,在不需要调用者提供事务环境的情况下都可以使用此级别.
能保证自己一定在事务环境的同时,兼容事务或非事务方法的调用.


##### 2.1.2.2 REQUIRES_NEW

>表示该方法会在一个新的事务环境中执行.

| 事务情况 | 子方法抛出异常父方法不捕获 | 子方法抛出异常父方法捕获并处理 | 父方法抛出异常 |
| :-------- | :-------- | :-------- | :-------- |
|父方法有事务 -> 子方法新建事务|两事务都回滚|子方法事务回滚|父方法事务回滚|
|父方法无事务 -> 子方法新建事务|子方法事务回滚|子方法事务回滚|无事务回滚|

###### 使用情况
需要子方法事务的成败不受父方法影响,也不能影响父方法事务的成败.

如果业务需求每接受到一次请求无论业务是否成功都要记录日志到数据库，如下图：

![充值处理](https://raw.githubusercontent.com/eva00rei00/PicStore/master/pic/20181204103347.png)

因为log()的操作不管扣款和创建订单成功与否都要生成日志，并且日志的操作成功与否不影响充值处理，所以log()方法的事务传播行为可以定义为:REQUIRES_NEW.

##### 2.1.2.3 NESTED

* **注意!JpaTransactionManager不支持此传播行为**

>表示该方法会以子事务的方式加入原有的事务环境,如果没有事务环境就会在一个新的事务环境中执行.
>NESTED是Spring所提供的一个特殊传播行为,嵌套子事务，实际上是借助jdbc的savepoint实现的，与外层父事务属于同一个事物。他的作用是如果子方法发生回滚则只会回滚至savepoint(子方法开始前),不会回滚整个事务.

| 事务情况 | 子方法抛出异常父方法不捕获 | 子方法抛出异常父方法捕获并处理 | 父方法抛出异常 |
| :-------- | :-------- | :-------- | :-------- |
|父方法有事务 -> 子方法嵌套事务|父子事务回滚|子事务回滚|父子事务回滚|
|父方法无事务 -> 子方法新建事务|子方法事务回滚|子方法事务回滚|无事务回滚|

**\* NESTED的回滚可以总结为，子事务回滚到savepoint，父事务可选择性回滚或者不回滚；父事务回滚子事务一定回滚。**

###### 使用情况

子方法的事务的成败要受父方法事务成败的影响,但是不能影响父方法事务.

在银行新增银行卡业务中，需要执行两个操作，一个是保存银行卡信息，一个是登记新创建的银行卡信息，其中登记银行卡信息成功与否不影响银行卡的创建。

![新增银行卡](https://raw.githubusercontent.com/eva00rei00/PicStore/master/pic/20181204103925.png)

由以上需求，我们可知对于regster()方法的事务传播行为，可以设置为NESTED，action()事务的回滚，regster()保存的信息就没意义，也就需要跟着回滚，而regster()的回滚不影响action()事务；insert()的事务传播行为可以设置为REQUIRED, MANDATORY，即insert()回滚事务，action()的事务必须跟着回滚。

##### 2.1.2.4 MANDATORY

>表示当前方法必须在一个原有的事务环境中运行，如果没有事务环境，将抛出异常
>> org.springframework.transaction.IllegalTransactionStateException: No existing transaction found for transaction marked with propagation 'mandatory'

| 事务情况 | 子方法抛出异常父方法不捕获 | 子方法抛出异常父方法捕获并处理 | 父方法抛出异常 |
| :-------- | :-------- | :-------- | :-------- |
|父方法有事务 -> 子方法加入事务|事务回滚|事务回滚(rollback only)|事务回滚|
|父方法无事务 -> 子方法抛出异常|不存在此情况|不存在此情况|不存在此情况|

这种方式适用于强制调用者必须在事务环境内调用此方法的情况.

###### 使用情况

在一个话费充值业务处理逻辑中，有如下图所示操作:

![充值处理](https://raw.githubusercontent.com/eva00rei00/PicStore/master/pic/20181204102826.png)

chargeHandle()方法有charger()和order()两个子方法,因为业务需要扣款操作和创建订单操作同成功或者失败，因此，charger()和order()的事务不能相互独立，需要包含在chargeHandle()的事务中；
通过以上需求，可以给charge()和order()的事务传播行为定义成：*MANDATORY* 确保父子三个方法一定在同一个事务环境中.
只要charge()或者order()抛出异常整个chargeHandle()都一起回滚，即使chargeHandle()捕获异常也没用，不允许提交事务。


##### 2.1.2.5 SUPPORTS

>表示该方法可以支持事务,会加入原有的事务环境,如果没有事务环境则以非事务执行.

| 事务情况 | 子方法抛出异常父方法不捕获 | 子方法抛出异常父方法捕获并处理 | 父方法抛出异常 |
| :-------- | :-------- | :-------- | :-------- |
|父方法有事务 -> 子方法加入事务|事务回滚|事务回滚(rollback only)|事务回滚|
|父方法无事务 -> 子方法无事务|无事务|无事务|无事务|

这种方式适用于需要兼容事务环境和非事务环境调用此方法的情况.
将是否运行在事务环境中的选择权交给父方法.

##### 2.1.2.6 NOT_SUPPORTED

>表示该方法不支持事务,不能运行在一个事务环境中,如果在事务环境中调用,事务将会被挂起,如果没有事务环境则以非事务执行.

| 事务情况 | 子方法抛出异常父方法不捕获 | 子方法抛出异常父方法捕获并处理 | 父方法抛出异常 |
| :-------- | :-------- | :-------- | :-------- |
|父方法有事务 -> 子方法无事务|父方法事务回滚|无事务回滚|父方法事务回滚|
|父方法无事务 -> 子方法无事务|无事务|无事务|无事务|

**\* 子方法使用NOT_SUPPORTED传播行为和不加事务注解的区别是当父方法原本有事务环境的情况下,NOT_SUPPORTED会把事务挂起,待子方法提交后再继续执行事务方法,而如果是子方法没有使用事务注解,那子方法的提交会和父方法事务一起提交.**

##### 2.1.2.7 NEVER

>表示该方法不能运行在一个事务环境中,否则将抛出异常,如果没有事务环境则以非事务执行.

| 事务情况 | 子方法抛出异常父方法不捕获 | 子方法抛出异常父方法捕获并处理 | 父方法抛出异常 |
| :-------- | :-------- | :-------- | :-------- |
|父方法有事务 -> 子方法抛出异常|不存在此情况|不存在此情况|不存在此情况|
|父方法无事务 -> 子方法无事务|无事务|无事务|无事务|

这种方式适用于强制调用者禁止在事务环境内调用此方法的情况.
###### 使用情况
在订单的售后处理中，更新完订单金额后，需要自动统计销售报表，如下图所示：

![售后处理](https://raw.githubusercontent.com/eva00rei00/PicStore/master/pic/20181204103714.png)

根据业务可知，售后是已经处理完订单的充值请求后的功能，是对订单的后续管理，统计报表report()方法耗时较长，因此，我们需要设置report()的事务传播行为为:PROPAGATION_NEVER,表示不适合在有事务的操作中调用，因为report()太耗时。

#### 2.1.3 传播行为使用时的注意事项

1. **需要特别注意的是 *NOT_SUPPORTED* , *NEVER* , *SUPPORTS* 这三种传播行为在使用时会出现方法以非事务方式执行的情况.如果以非事务执行的情况下即使抛出异常,之前的数据库操作也不会回滚,所以一定确认方法在没有异常回滚的情况下可以正常运行时才考虑使用以上三种传播行为.(简单说就是如果方法中有多条数据库操作,千万不要使用以上三种传播行为)**

2. **当传播行为为 *REQUIRED* , *MANDATORY* ,或 *SUPPORTS* 这三者之一,并且是加入当前事务执行的情况,如果子方法抛出了异常回滚,事务就会被标记为rollback only.无论父方法是否捕获处理异常,都只能回滚,如果父方法提交事务会抛出运行时异常TransactionException.**
  **这可能与我们的直觉相悖,明明父方法没有抛出异常,为什么无法提交呢?其实这很好理解,父子两个方法本来就是同属于同一个事务,不会出现事务部分回滚的情况.**

3. **使用默认配置 *REQUIRED* 以外的传播行为时,请在方法doc注释里详细说明原因并告知可能的调用结果,避免造成误会.**

4. ***NESTED* 与 *REQUIRES_NEW* 之间的区别是在父方法有事务的情况下 NESTED 开始一个 嵌套的事务,  它是已经存在事务的一个子事务. 嵌套事务开始执行时,  它将取得一个 savepoint. 如果这个嵌套事务失败, 我们将回滚到此 savepoint. 嵌套事务是外部事务的一部分, 只有外部事务结束后它才会被提交. 
REQUIRES_NEW 会建立一个新事物. 这个事务将被完全 commited 或 rolled back 而不依赖于外部事务, 它拥有自己的隔离范围, 自己的锁, 等等. 当新事务开始执行时, 原有事务将被挂起直到事务结束. 
因为子事务实际上于父事务是同一个事务,而新事务与原事务没有任何关联.所以有下表:**

| 在父方法有事务的情况下子事务的传播行为 | 子方法抛出异常父方法不捕获 | 子方法抛出异常父方法捕获并处理 | 父方法抛出异常 |
| :-------- | :-------- | :-------- | :-------- |
|NESTED|父子事务回滚|子事务回滚|父子事务回滚|
|REQUIRES_NEW|两事务都回滚|子方法事务回滚|父方法事务回滚|

5. **太过耗时的方法如无必要不要使用事务,最好使用 *NEVER* , *SUPPORTS* 确保不在事务环境中执行,因为会大量占用数据库资源,阻塞其他事务的执行,并会大大增加死锁发生的概率.**

### 2.2 isolation事务隔离级别

> TransactionDefinition中定义了4种隔离级别,默认值ISOLATION_DEFAULT会使用底层数据库默认的隔离级别。

``` java
    /**
    * 使用底层数据库默认隔离级别
    * MySQL默认隔离级别为REPEATABLE_READ
    * Sql Server , Oracle默认隔离级别为READ_COMMITTED
    */
    int ISOLATION_DEFAULT = -1;
    /**
    * 读未提交
    * 当前事务可以读取另一个事务修改但还没有提交的数据。
    * 脏读 √ 不可重复读 √ 幻读 √
    */
    int ISOLATION_READ_UNCOMMITTED = Connection.TRANSACTION_READ_UNCOMMITTED;
    /**
    * 读已提交
    * 当前事务只能读取到其他事务提交的数据，未提交的数据读不到。
    * 脏读 × 不可重复读 √ 幻读 √
    */
    int ISOLATION_READ_COMMITTED = Connection.TRANSACTION_READ_COMMITTED;
    /**
    * 可重复读
    * 当前事务在整个过程中可以多次重复执行某个查询，并且每次返回的记录都相同。
    * 脏读 × 不可重复读 × 幻读 √ (对于MySQLInnoDB:脏读 × 不可重复读 × 幻读 ×)
    */
    int ISOLATION_REPEATABLE_READ = Connection.TRANSACTION_REPEATABLE_READ;
    /**
    * 串行化
    * 所有的事务依次逐个执行,其他会话对该表的写操作将被挂起。
    * 脏读 × 不可重复读 × 幻读 ×
    */
    int ISOLATION_SERIALIZABLE = Connection.TRANSACTION_SERIALIZABLE;
```

#### 2.2.2 不可重复读和幻读的区别

不可重复读和幻读都是在同一个事务中两次查询到不同的结果,他俩的区别是不可重复读变化的是字段里属性的值,幻读变化得是字段的数量.

##### 不可重复读

不可重复读指的是一个事务多次读取同一字段的属性发生了变化,根本原因是同一字段在两次查询之间被另一并行的事务修改了

|事务A|事务B|
|:---|:---|
|开始事务|开始事务|
|查询余额:1000元||
||查询余额:1000元|
||取出1000元|
||提交事务|
|查询余额:0元|
事务A的两次查询结果与预期值不符,因为并行的事务A也修改了余额.

##### 幻读

幻读指的是一个事务以某一条件多次查询多条记录时,前后查询到的字段数量发生了变化.根本原因是在两次查询之间并行的其他事务的某些操作改变了符合条件字段的数量导致的.

|事务A(删除所有finish = 1 的记录)|事务B(修改某记录状态 finish 0 -> 1)|
|:---|:---|
|开始事务|开始事务|
|删除所有 finish = 1 的记录||
||修改一个记录 finish 0 -> 1|
||提交事务|
|查询 finish = 1 的记录,还能查到一条||

事务A明明已经删除了全部的finish = 1记录,但是再次查询居然还有符合条件的记录就像是出现幻觉了一样.

|事务A(查询并插入记录)|事务B(插入记录)|结果|
|:---|:---|:--|
|开始事务|开始事务|
|select * from t where id>3;||return NULL;|
||insert into t values(4);|1 row affected|
||提交事务|
|insert into t values(4);||Error : duplicate key!|

事务A明明查了id>3为空集，insert id=4却出现了PK冲突

#### 2.2.3 四种隔离级别在MySQL InnoDB引擎的实现方式

##### 读未提交(Read Uncommitted)

>这种事务隔离级别下，select语句不加锁。

此时，可能读取到不一致的数据，即“读脏”。这是并发最高，一致性最差的隔离级别。

##### 串行化(Serializable)

>这种事务的隔离级别下，所有select语句都会被隐式的转化为select ... in share mode.

这可能导致，如果有未提交的事务正在修改某些行，所有读取这些行的select都会被阻塞住。

这是一致性最好的，但并发性最差的隔离级别。

###### 在互联网大数据量，高并发量的场景下，几乎不会使用上述两种隔离级别。

##### 可重复读(Repeated Read, RR)

这是InnoDB默认的隔离级别，,可以防止**幻读**与不可重复读. 在RR下：

1. 普通的select使用快照读(snapshot read)，这是一种不加锁的一致性读(Consistent Nonlocking Read)，底层使用MVCC来实现.

2. 加锁的select(select ... in share mode / select ... for update), update, delete等语句是当前读，要加锁.它们的锁，依赖于它们是否在唯一索引(unique index)上使用了唯一的查询条件(unique search condition)，或者范围查询条件(range-type search condition)：
* 在唯一索引上使用唯一的查询条件，会使用记录锁(record lock)，而不会封锁记录之间的间隔，即不会使用间隙锁(gap lock)与临键锁(next-key lock),因为只读一条记录不会出现幻读.
* 范围查询条件，会使用间隙锁与临键锁，锁住索引记录之间的范围，避免范围间插入记录，以避免产生幻读

##### 读已提交(Read Committed, RC)

这是互联网最常用的隔离级别.在RC下：

1. 普通读是快照读(也是MVCC,但与RR级别的规则不同,不能防止幻读和不可重复读)；

2. 加锁的select, update, delete等语句是当前读，除了在外键约束检查(foreign-key constraint checking)以及重复键检查(duplicate-key checking)时会封锁区间，其他时刻都只使用记录锁；
**\* 此时，其他事务的插入依然可以执行，就可能导致，读取到幻影记录。**

#### 2.2.4 在代码中使用数据库隔离级别的建议(MySQL InnoDB)

* 那么到底该使用哪种隔离级别?
  我觉得MySql默认RR就挺好,不论如何别用RU和Serializable,可在以下情况可改为使用RC级别优化效率.
  1. 表字段在业务逻辑上没有并发的可能.(例如个人信息表)
  2. 不会出现幻读和不可重复读或其产生的影响可控.
* InnoDB中的行锁(包括记录锁,间隙锁与临键锁)的实现依赖于索引，一旦某个加锁操作没有使用到索引，那么该锁就会退化为表锁。而我们一般不会手动创建索引,所以此时主键之外的锁查询皆为表锁,频繁的update和delete是非常低效的(唯一索引上使用唯一的查询条件除外)
  1. 为每个属性建立索引
  2. insert,update,delete语句尽量使用主键做唯一条件,代码层面尽量减少每个事务中的数据库操作逻辑,让事务尽可能小

## 3. 其他@Transactional 注解使用时需注意的问题
1. **避免 AOP 自调用** 
在 Spring 的 AOP 代理下，只有目标方法由外部调用，目标方法才由 Spring 生成的代理对象来管理。若同一类中的方法相互调用，@Transactional注解不会生效,如果要在同一类中调用事务方法需要通过注入的对象调用。
2. **不要在接口里使用@Transactional注解**
在接口上使用 @Transactional 注解，只能当你设置了基于接口的代理时它才生效。因为注解是不能继承的，所以如果在程序中直接使用了接口的实现类对象,事务注解将不会生效.
并且如果在开启事务注解驱动\<tx:annotation-driven proxy-target-class="true"/>开启了proxy-target-class属性,基于接口的注解将不会被解析.
3. **@Transactional 注解应用到 public 可见度的方法上才有效**
如果你在 protected、private 或者 package-visible 的方法上使用 @Transactional 注解，                                                               会报错，但事务不会生效。
4. **关于只读事务**
* 如果方法中只有查询语句指定readOnly=true,可以提升数据库查询效率.
* 只读事务中如果有增删改操作将被忽略.
* 如果是子方法加入当前事务环境的方式执行,如果父方法没有readOnly=true,子方法设置为true也不会生效.
* 如果方法最终没有运行在事务环境里(例如传播级别设置为NOT_SUPPORTED)readOnly=true不会生效.



## 参考文章
* [解惑 spring 嵌套事务](http://www.iteye.com/topic/35907)
* [@transactional 使用注意事宜](https://my.oschina.net/u/2349605/blog/413563)
* [spring 中常用的两种事务配置方式以及事务的传播性、隔离级别](https://blog.csdn.net/qh_java/article/details/51811533)
* [spring 事务传播行为实例分析](https://blog.csdn.net/pml18710973036/article/details/58607148)
* [Spring Transactional ReadOnly 和 不加Transactional的区别](https://segmentfault.com/q/1010000008383351)
* [4种事务的隔离级别，InnoDB如何巧妙实现？](https://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651961498&idx=1&sn=058097f882ff9d32f5cdf7922644d083&chksm=bd2d0d468a5a845026b7d2c211330a6bc7e9ebdaa92f8060265f60ca0b166f8957cbf3b0182c&scene=21#wechat_redirect)
* [MySQL的锁机制 - 记录锁、间隙锁、临键锁](https://zhuanlan.zhihu.com/p/48269420)
