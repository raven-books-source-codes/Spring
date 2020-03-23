[toc]

![](C:\Users\Raven\Pictures\blog\fe80000239b95cab103f.png)

## 1. 问题引入：

先看下面这个代码：

```java
    @Override
    public void transfer(String sourceName, String destName, Float money) {
        Account source = accountDao.findAccountByName(sourceName);
        Account dest = accountDao.findAccountByName(destName);

        source.setMoney(source.getMoney() - money);
        dest.setMoney(dest.getMoney()+money);

        accountDao.updateAccount(source);
        // 如果这里执行 1/0 排除异常
        accountDao.updateAccount(dest);
    }
```

如果执行了1/0，排除异常，则最后一个update得不到执行，数据库无法保持一致性。

![](https://api.superbed.cn/item/5e77704f5c56091129f11a26.png)

### 1.2 问题解决

使用ThreadLocal将当前线程和数据源连接绑定，加上事务管理控制。下面是所有源代码：

ConnectionUtils:

```java
public class ConnectionUtils {
    private ThreadLocal<Connection> tl = new ThreadLocal<>();

    // 数据源，等待spring注入
    private DataSource dataSource;

    public void setDataSource(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public Connection getConnection(){
        Connection conn = tl.get();
        try{
            if(conn == null){
                conn = dataSource.getConnection();
                tl.set(conn);
            }
        }catch (SQLException e){
            e.printStackTrace();
        }
        return conn;
    }

    public void removeConnection(){
        tl.remove();
    }
}
```

事务控制：

```java
public class TransactionManager {
    // 等待spring注入
    private ConnectionUtils connectionUtils;


    public void setConnectionUtils(ConnectionUtils connectionUtils) {
        this.connectionUtils = connectionUtils;
    }

    public void beginTransaction(){
        try {
            connectionUtils.getConnection().setAutoCommit(false);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    public void commit(){
        try {
            connectionUtils.getConnection().commit();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
    public void rollback(){
        try {
            connectionUtils.getConnection().rollback();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
    public void release(){
        try {
            connectionUtils.getConnection().close();
            // 注意释放thradlocal
            connectionUtils.removeConnection();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

AccountDaoImpl:

主要是将queryRunner的执行connection替换为threadlocal中的connection:

```java
public class AccountDaoImpl implements IAccountDao {

    private QueryRunner runner;
    private ConnectionUtils connectionUtils;

    public void setRunner(QueryRunner runner) {
        this.runner = runner;
    }

    public void setConnectionUtils(ConnectionUtils connectionUtils) {
        this.connectionUtils = connectionUtils;
    }

    @Override
    public List<Account> findAllAccounts() {
        List<Account> accounts;
        try{
          accounts =  runner.query(connectionUtils.getConnection(),"select * from account",new BeanListHandler<>(Account.class));
        }catch(Exception e){
            throw new RuntimeException(e.getMessage());
        }
        return accounts;
    }

    @Override
    public Account findAccountById(Integer id) {
        try{
            return runner.query(connectionUtils.getConnection(),"select * from account where id=?",new BeanHandler<>(Account.class),id);
        }catch(Exception e){
            throw new RuntimeException(e.getMessage());
        }
    }

    @Override
    public void saveAccount(Account account) {
        try {
            runner.update(connectionUtils.getConnection(),"insert into account (id,name,money) values (null,?,?)",account.getName(),account.getMoney());
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void updateAccount(Account account) {
        try {
            runner.update(connectionUtils.getConnection(),"update account set name=?, money=? where id=?",
                    account.getName(),account.getMoney(),account.getId());
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void deleteAccountById(Integer id) {
        try {
            runner.update(connectionUtils.getConnection(),"delete from account where id=?",id);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    @Override
    public Account findAccountByName(String name) {
        List<Account> accounts;
        try{
            accounts =  runner.query(connectionUtils.getConnection(),"select * from account where name=?",new BeanListHandler<>(Account.class),name);
        }catch(Exception e){
            throw new RuntimeException(e.getMessage());
        }
        if(accounts == null || accounts.size() == 0){
            return null;
        }else if(accounts.size() > 1){
            throw new RuntimeException("find more than 1 account matching required");
        }
        return accounts.get(0);
    }
}
```

最后修改service:

```java
@Override
public void transfer(String sourceName, String destName, Float money) {
    try{
        txManager.beginTransaction();
        Account source = accountDao.findAccountByName(sourceName);
        Account dest = accountDao.findAccountByName(destName);
        source.setMoney(source.getMoney() - money);
        dest.setMoney(dest.getMoney()+money);
        accountDao.updateAccount(source);
        int i = 1/0;
        accountDao.updateAccount(dest);
        txManager.commit();
    }catch(Exception e){
        txManager.rollback();
        throw new RuntimeException(e);
    }finally {
        txManager.release();
    }
}
```

可以看到方法变得非常臃肿，没调用一个业务层逻辑，只要业务层调用到了持久层，都会使用到事务管理。

## 2. 动态代理

打个比方，客户-- 经销商 -- 产家

![](C:\Users\Raven\Pictures\blog\fee10000d056297a082a.png)

特点：字节码随用随创建，随用随加载

作用：不修改源码的基础上对方法增强

	- 基于接口的动态代理
	- 基于子类的动态代理

### 2.1 基于接口的动态代理

涉及的类：Poxy

提供者：JDk官方

​	如何创建代理对象：

​	使用Proxy类中的 newProxyInstance方法

创建代理对象的要求:被代理类最少实现一个接口，如果没有则不能使用

newProxyInstance方法的参数

- CLassloader：类加载器
  它是用于加载代理对象字节码的。和被代理对象使用相同的类加载器。固定写法
- Class[] 是用于让代理对象和被代理对象有相同方法。固定写法
- Invocatlonhandler, 提供增强的代码
  它是让我们写如何代理。我们一般都是写一个该接口的实现类，通常情况下都是匿名内部类，但不是必须
  此接口的实现类都是谁用谁写。

```java
    // 使用动态代理
        Producer producer = new Producer();

        IProducer iProducer = (IProducer) Proxy.newProxyInstance(producer.getClass().getClassLoader(),
                producer.getClass().getInterfaces(), new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        System.out.println(proxy);
                        // 经销商加强
                        Object result = null;
                        if ("saleProduct".equals(method.getName())) {
                            result = method.invoke(producer, ((Float) args[0]) * 0.8f);
                        }
                        return result;
                    }
                });
        iProducer.saleProduct(123f);
```

### 2.2 基于子类的动态代理 

基于子类的动态代理

​		涉及的类： Enhancer

​		提供者：第三方cgib库

如何创建代理对象：

​		使用 Enhancer类中的 create方法

创建代理对象的要求，被代理对象不能是final

create方法的参数

​			Class：字节码，它是用于指定被代理对象的字节码。

​		Callback用于提供增强的代码，它是让我们写如何代理。我们一般都是些一个该接口的实现类，通常情况下都是匿名内部类，但不是必须的。

此接囗的实现类都是谁用谁写

```java
  Producer producer = new Producer();

        // 使用子类的动态代理
        Producer producerProxy = (Producer) Enhancer.create(producer.getClass(),
                new MethodInterceptor() {
                    @Override
                    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                        // 经销商加强
                        Object result = null;
                        if ("saleProduct".equals(method.getName())) {
                            result = method.invoke(producer, ((Float) objects[0]) * 0.8f);
                        }
                        return result;
                    }
                });

        producerProxy.saleProduct(123f);
```

## 3. AOP

### 3.1 什么是AOP

AOP：全称是Aspect Oriented Programming即：面向切面编程。

![](https://api.superbed.cn/item/5e77707c5c56091129f12f62.png)

简单的说它就是把我们程序重复的代码抽取出来，在需要执行的时候，使用动态代理的技术，在不修改源码的基础上，对我们的已有方法进行增强。

![](C:\Users\Raven\Pictures\blog\H785d7bfd3def4243b5ca82a31dc688503.png)

### 3.2  AOP的作用与优势

作用： 在程序运行期间，不修改源码对已有方法进行增强。 

优势： 1、减少重复代码 2、提高开发效率 3、维护方便

AOP采用动态代理技术。

### 3.3 AOP的相关术语

#### Joinpoint(连接点):

所谓连接点是指那些被拦截到的点。在spring中,这些点指的是方法,因为spring只支持方法类型的连接点。**即所有的业务接口方法。**

```java
public interface IAccountService {
    List<Account> findAllAccounts();

    Account findAccountById(Integer id);

     void saveAccount(Account account);

     void updateAccount(Account account);

     void deleteAccountById(Integer id);

     void transfer(String sourceName,String destName, Float money);
     
     void test(); // 这个是连接点，但是不是切入点
}
```

#### Pointcut(切入点): 

所谓切入点是指我们要对哪些Joinpoint进行拦截的定义。

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if("test".equals(method.getName())){
        return null;
    }
```

#### Advice(通知/增强): 

所谓通知是指拦截到Joinpoint之后所要做的事情就是通知。 通知的类型：前置通知,后置通知,异常通知,最终通知,环绕通知。

![](C:\Users\Raven\Pictures\blog\ff8e0000d37499e29575-1584885998367.jpg)

#### Introduction(引介): 

引介是一种特殊的通知在不修改类代码的前提下, Introduction可以在运行期为类动态地添加一些方法或Field。

####  Target(目标对象): 

代理的目标对象。 

#### Weaving(织入): 

是指把增强应用到目标对象来创建新的代理对象的过程。 spring采用动态代理织入，而AspectJ采用编译期织入和类装载期织入。

####  Proxy（代理）: 

一个类被AOP织入增强后，就产生一个结果代理类。

####  Aspect(切面): 

是切入点和通知（引介）的结合。

### 3.4 AOP的xml配置

准备材料：

1. 切入点，Service的接口方法
2. 通知类，Logger

预备实现，在每个切入方法前自动执行Logger。

如：

![](C:\Users\Raven\Pictures\blog\fe57000121ef45e679f4.png)

配置流程：

```
spring中的基于xml的aop配置
1. 把通知的bean也交给spring来管理
2. 使用aop:config标签，表明开始aop的配置
3. 使用aop:aspect表明配置切面
        id属性：是给切面提供一个唯一标识
        ref属性：是指定通知类bean的Id
4. 在aop：aspect的内部，使用对应的标签来配置通知的类型
        我们现在的示例是让printLog在切入点之前执行（前置通知）
          method属性：用于指定 Logger类中哪个方法是前置通知
          pointcut属性：用于指定切入点表达式，该表达式的含义指的是对业务层中哪些方法增强
        切入点表达式的写法：
           关键字： execution（表达式）
           访问修饰符返回值包名。包名，包名。类名。方法名（参数列表）
           标准的
           public void com. itheima service impl. Account ServiceImpl saveAccountO    
	  全通配表达式：
                    1. 访问修饰符可以省略
                    2. 返回值可以用通配符，表示任意返回值
                    3. 包名可以用通配符，*.. 表示当前包和子包
                    4. 类名和方法可以用通配符, *.*
                    5. 参数 - 基本类型直接写，引用类型用包名.类名
                        或者直接写..代表有无参数都可以。有参数也是任意类型、
                    所以全通配为 * *..*.*(..)
        开发中，不要写全通配，一般是切到业务层实现类的所有方法
                 	* service..*.*(..)
```

```xml
<bean id="accountService" class="service.impl.AccountServiceImpl"/>
<!--
        配置Logger
-->
    <bean id="logger" class="utils.Logger"/>

<!--
        配置aop
-->
    <aop:config>
        <aop:aspect id="logAdvice" ref="logger">
            <aop:before method="printLog"
                        pointcut="execution(public void service.IAccountService.saveAccount())"/>
        </aop:aspect>
    </aop:config>
```

注意切入点表达式需要依赖aspectjweaver：

```xml
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>1.9.5</version>
</dependency>
```

### 3.5  其余通知的配置

```xml
  <aop:config>
        <aop:aspect id="logAdvice" ref="logger">
<!--
            可以添加多个相同method的标签
-->
<!--            切入点前执行-->
            <aop:before method="beforeLog"
                        pointcut="execution(* service..*.*(..))"/>
<!--            切入点后执行-->
            <aop:after-returning method="afterReturnLog"
                        pointcut="execution(* service..*.*(..))"/>
<!--            发生异常执行-->
            <aop:after-throwing method="afterThrowLog"
                        pointcut="execution(* service..*.*(..))"/>
<!--            最终一定执行-->
            <aop:after method="afterLog"
                        pointcut="execution(* service..*.*(..))"/>
        </aop:aspect>
```

注意后置通知和异常通知只能执行一个。

 或者：

```xml
   <aop:config>
<!--        放在aop:config中，就可以在所有切面使用-->
        <aop:pointcut id="pt1" expression="execution(* service..*.*(..))"/>
        <aop:aspect id="logAdvice" ref="logger">
<!--
            可以添加多个相同method的标签
-->
<!--            切入点前执行-->
            <aop:before method="beforeLog" pointcut-ref="pt1"/>
<!--            切入点后执行-->
            <aop:after-returning method="afterReturnLog"
                                 pointcut-ref="pt1"/>
<!--            发生异常执行-->
            <aop:after-throwing method="afterThrowLog"
                                pointcut-ref="pt1"/>
<!--            最终一定执行-->
            <aop:after method="afterLog"
                       pointcut-ref="pt1"/>

<!--            如果放在aop:aspect标签内，则pointcut只能在当前切面使用-->
<!--            <aop:pointcut id="pt1" expression="execution(* service..*.*(..))"/>-->
        </aop:aspect>

    </aop:config>
```

#### 3.5.2 环绕通知

```
环绕通知
问题：
	当我们配置了环绕通知之后，切入点方法没有执行，而通知方法执行了。
分析：
	通过对比动态代理中的环绕通知代码，发现动态代理的环绕通知有明确的切入点方法调用，而我们的代码中没有。
解决：
	Spring框架为我们提供了一个接口： ProceedingJoinPoint.该接口有一个方法 proceed（），此方法就相当于明确调用切入点方法。
	该接口可以作为环绕通知的方法参数，在程序执行时， spring框架会为我们提供该接口的实现类
spring中的环绕通知：
	它是 spring框架为我们提供的一种可以在代码中手动控制增强方法何时执行的方式
```

```java
/**
 * 环绕通知
 * 问题：配置了环绕通知后，切入点的方法没有执行，而通知方法执行了。
 * 分析：
 *
 */
public Object aroundLog(ProceedingJoinPoint pjp) {
    Object obj = null;
    try {
        System.out.println("环绕通知");
        obj = pjp.proceed(pjp.getArgs());
        return obj;
    } catch (Throwable t) {
        throw new RuntimeException(t);
    }
}
```

### 3.6 基于注解的aop配置

对于通知类，首先将它加入spring ioc容器

然后用@Aspect注解声明它是一个切面

接着定义 aop 的切入点。

使用：

	1. @Before 前置通知
 	2. @AfterReturning后置通知
 	3. @AfterThrowing异常通知
 	4. @After 最终通知

```java
@Component("logger")
@Aspect
public class Logger {

    @Pointcut("execution(* service.impl.*.*(..))")
    public void pointcut1(){}

    {
        System.out.println("load Logger class");
    }

    /**
     * 打印日志，计划让其切入点方法执行之前执行（切入点方法就是业务层方法）
     */
    @Before("pointcut1()")
    public void beforeLog(){
        System.out.println("before log...");
    }

    @AfterReturning("pointcut1()")
    public void afterReturnLog(){
        System.out.println("afterReturn log");
    }

    @AfterThrowing("pointcut1()")
    public void afterThrowLog(){
        System.out.println("afterThrow log");
    }

    @After("pointcut1()")
    public void afterLog(){
        System.out.println("after log");
    }

    /**
     * 环绕通知
     * 问题：配置了环绕通知后，切入点的方法没有执行，而通知方法执行了。
     * 分析：
     *
     */
//    @Around("pointcut1()")
    public Object aroundLog(ProceedingJoinPoint pjp) {
        Object obj = null;
        try {
            System.out.println("环绕通知");
            obj = pjp.proceed(pjp.getArgs());
            return obj;
        } catch (Throwable t) {
            throw new RuntimeException(t);
        }
    }
```

**注意需要开启EnableAspectJAutoProxy**

1. 注解方式开启：

   ```java
   @EnableAspectJAutoProxy
   public class SpringConfiguration {
   }
   ```

2. xml方式开启

   ```xml
       <aop:aspectj-autoproxy proxy-target-class="true"/>
   ```

   