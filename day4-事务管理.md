[toc]

![image-20200323094411778](https://i.loli.net/2020/03/23/3F54STfPgEi2ABa.png)

## 1. JDBCTemplate

![](https://i.loli.net/2020/03/23/KO1uT6GUtCpFrWs.jpg)

它是spring框架中提供的一个对象，是对原始Jdbc API对象的简单封装。spring框架为我们提供了很多的操作模板类。

操作关系型数据的： JdbcTemplate HibernateTemplate 

操作nosql数据库的： RedisTemplate

操作消息队列的： JmsTemplate

### 1.2 jdbcTemplate的使用方法

```java
// add
jt.update("insert into account (name,money) values ('haha',100)");

// delete
jt.update("delete from account where id=?",7);

// update
jt.update("update account set name='ddd' where id=?",5);

// select
List<Account> accounts = jt.query("select * from account where name like '___'",
        new BeanPropertyRowMapper<Account>(Account.class));
if(accounts != null){
    accounts.forEach(System.out::println);
}
```

### 1.3 jdbcSupoort父类

这个类作为父类，包含了jdbcTemplate成员，这样在子类中就可以不用写重复的jdbcTemplate对象了。举个例子：

![image-20200323105125136](https://i.loli.net/2020/03/23/lviPoYdBSxjs5FE.png)

但是这种方式，只能通过xml方式注入。

```xml
<bean id="accountDao" class="dao.impl.AccountDaoImpl">
    <property name="dataSource" ref="dataSource"/>
</bean>
或者
  <bean id="accountDao" class="dao.impl.AccountDaoImpl">
        <property name="jdbcTemplate" ref="jdbcTemplate"/>
    </bean>
```

## 2. Spring中的事务控制

### 2.1 Spring事务控制我们要明确的

第一：JavaEE体系进行分层开发，事务处理位于业务层，Spring提供了分层设计业务层的事务处理解决方案。 

第二：spring框架为我们提供了一组事务控制的接口。具体在后面的第二小节介绍。这组接口是在spring-tx-5.0.2.RELEASE.jar中。

 第三：spring的事务控制都是基于AOP的，它既可以使用编程的方式实现，也可以使用配置的方式实现。我们学习的重点是使用配置的方式实现。

### 2.2 Spring中事务控制的API介绍

#### 2.2.1 PlatformTransactionManager

此接口是spring的事务管理器，它里面提供了我们常用的操作事务的方法，如下图：

![image-20200323113106950](https://i.loli.net/2020/03/23/iC5JHxFc37gVjZt.png)

我们在开发中都是使用它的实现类，如下图：

真正管理事务的对象 org.springframework.jdbc.datasource.DataSourceTransactionManager 使用Spring JDBC或iBatis 进行持久化数据时使用

 org.springframework.orm.hibernate5.HibernateTransactionManager 使用Hibernate版本进行持久化数据时使用

#### 2.2.2 TransactionDefinition

它是事务的定义信息对象，里面有如下方法：

![image-20200323113147071](https://i.loli.net/2020/03/23/3c4PLiwTSJxXbQI.png)

##### 2.2.2.1 事务的隔离级别

![image-20200323113201807](https://i.loli.net/2020/03/23/tFChfjQyncwIRoS.png)

##### 2.2.2.2 事务的传播行为

REQUIRED:如果当前没有事务，就新建一个事务，如果已经存在一个事务中，加入到这个事务中。一般的选择（默认值） 

SUPPORTS:支持当前事务，如果当前没有事务，就以非事务方式执行（没有事务）

 MANDATORY：使用当前的事务，如果当前没有事务，就抛出异常 REQUERS_NEW:新建事务，如果当前在事务中，把当前事务挂起。

 NOT_SUPPORTED:以非事务方式执行操作，如果当前存在事务，就把当前事务挂起

 NEVER:以非事务方式运行，如果当前存在事务，抛出异常

 NESTED:如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行REQUIRED类似的操作。

##### 2.2.2.3 超时时间

默认值是-1，没有超时限制。如果有，以秒为单位进行设置。

##### 2.2.2.4 是否是只读事务

建议查询时设置为只读。

#### 2.2.3 TransactionStatus

此接口提供的是事务具体的运行状态，方法介绍如下图：

![image-20200323113256819](https://i.loli.net/2020/03/23/gnoI9baZDxmH5AR.png)

## 3. Spring的声明式事务控制配置

### 3.1 基于xml

```
 配置事务管理器
 1、 导入tx命名空间
 2、 配置事务管理器
 3、 配置通知
     此时我们需要导入的约束tx名称空间和约束，同时也需要aop的
     使用tx： advice标签配置事务通知
         id：给事务通知起一个唯一标识
         transaction-manager：transaction- manager：给事务通知提供一个事务管理器引用
     配置AOP中的通用切入点表达式
 4、 建立切入点和通知之间关系
 5、 事务的属性
         isolation：用于指定的平隔离级别。默认值是 DEFAULT，表示使用效据库的默认隔离级
         propagation：用于指定
         是否只读。只有查询方法才能设置为true.默认值是 false，表示评今边
         播行为。獸认值是 REQUIRED，表示一定会有辜务，增删改的选择。查询方法可以选择 SUPPORTS.
         read-only：用于
         timeout：用于指定辜务的超时时间，默认值是-1，表示永不超时。如果指定了数值，以秒为单位。
         rol1back-for：用于指定一个异常，当产生该异常时，回滚
         不回滚。没有认值。表示任何异常都回滚
         no- rollback-fon：用于指定一个异帘，当产生该异常时，辜务不回滚，产生其他异帘时辜务回滚。没有默认值。表示任何异帘都回滚。
```

一个示例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        https://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <!-- 配置业务层-->
    <bean id="accountService" class="service.impl.AccountServiceImpl">
        <property name="accountDao" ref="accountDao"/>
    </bean>

    <!-- 配置账户的持久层-->
    <bean id="accountDao" class="dao.impl.AccountDaoImpl">
        <property name="dataSource" ref="dataSource"/>
    </bean>


    <!-- 配置数据源-->
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost:3306/spring?serverTimezone=UTC"/>
        <property name="username" value="root"/>
        <property name="password" value="0512"/>
    </bean>

<!--
    配置管理器
-->
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>

<!--
	配置通知
-->

    <tx:advice id="txAdvice" transaction-manager="transactionManager">
        <!-- 配置属性 -->
        <tx:attributes>
            <tx:method name="*" propagation="REQUIRED" read-only="false"/>
            <tx:method name="find*" propagation="SUPPORTS" read-only="true"/>
        </tx:attributes>
    </tx:advice>

    <!-- 配置AOP -->
    <aop:config>
        <aop:pointcut id="pt1" expression="execution(* service..*.*(..))"/>
        <!--     建立关系-->
        <aop:advisor advice-ref="txAdvice" pointcut-ref="pt1"/>
    </aop:config>

</beans>
```

### 3.2 基于注解

```
1、配置事务管理器
2、开始spring的注解事务支持
3、在需要事务支持的地方使用@Transacational注解
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        https://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <context:component-scan base-package="service dao"/>

    <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
        <property name="dataSource" ref="dataSource"/>
    </bean>


    <!-- 配置数据源-->
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost:3306/spring?serverTimezone=UTC"></property>
        <property name="username" value="root"></property>
        <property name="password" value="0512"></property>
    </bean>


    <!-- spring中基于注解的声明式事务控制配置步骤
    1、配置事务管理器
    2、开始spring的注解事务支持
    3、在需要事务支持的地方使用@Transacational注解
 -->


    <bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <tx:annotation-driven transaction-manager="txManager"/>

</beans>
```

```java
@Transactional(propagation = Propagation.REQUIRED, readOnly = true)
@Service("accountService")
public class AccountServiceImpl implements IAccountService {

    @Autowired
    private IAccountDao accountDao;

    @Override
    public Account findAccountById(Integer accountId) {
        return accountDao.findAccountById(accountId);

    }


    @Override
    public void transfer(String sourceName, String targetName, Float money) {
        System.out.println("transfer....");
            //2.1根据名称查询转出账户
            Account source = accountDao.findAccountByName(sourceName);
            //2.2根据名称查询转入账户
            Account target = accountDao.findAccountByName(targetName);
            //2.3转出账户减钱
            source.setMoney(source.getMoney()-money);
            //2.4转入账户加钱
            target.setMoney(target.getMoney()+money);
            //2.5更新转出账户
            accountDao.updateAccount(source);

            int i=1/0;

            //2.6更新转入账户
            accountDao.updateAccount(target);
    }
}
```

