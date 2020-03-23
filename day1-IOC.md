[toc]

## 1. Spring是什么？

Spring是分层的Java SE/EE应用 full-stack轻量级开源框架，以IoC（Inverse Of Control：反转控制）和AOP（Aspect Oriented Programming：面向切面编程）为内核，提供了展现层Spring MVC和持久层Spring JDBC以及业务层事务管理等众多的企业级应用技术，还能整合开源世界众多著名的第三方框架和类库，逐渐成为使用最多的Java EE企业应用开源框架。

## 2. Spring的架构

![image-20200320213408167](https://i.loli.net/2020/03/21/ztNyUM9Cw3s6cgv.png)

## 3. 耦合

```java
/**
 * @author Raven
 * @version 1.0
 * @date 2020/3/20 21:36
 *
 * 程序的耦合：
 *  耦合：程序间的依赖关系
 *         包括：
 *              类之间的依赖
 *              方法间的依赖
 *  解耦：
 *      降低程序间的依赖关系
 *  实际开发中，
 *      应该做到，编译器不依赖，运行时才依赖。
 *  解耦的思路：
 *      第一步，使用反射来创建对象，而避免使用new关键字
 *      第二步，通过读取配置文件来获取要创建的对象全限定类名
 */
public class JDBCDemo {
    public void test() throws SQLException, ClassNotFoundException {
        //jdbc
        //1. 注册驱动
//        DriverManager.registerDriver(new Driver());
        //  通过反射降低耦合
        Class.forName("com.mysql.jdbc.Driver");
        //2. 连接数据库
        Connection conn = DriverManager.getConnection(
                "jdbc:mysql://localhost:3306/spring?serverTimezone=UTC",
                "root","0512"
        );
        //3. 获取操作数据库的预处理对象
        PreparedStatement pstm = conn.prepareStatement("select * from account");
        //4. 执行sql语句
        ResultSet rs = pstm.executeQuery();
        //5. 查询
        while(rs.next()){
            System.out.println(rs.getString("name"));
        }
        // 6. 释放资源
        rs.close();
        pstm.close();
        conn.close();
    }

    public static void main(String[] args) throws SQLException, ClassNotFoundException {
        JDBCDemo jdbc = new JDBCDemo();
        jdbc.test();
    }
}
```

### 3.1 耦合的分析以及如何解耦

程序的耦合：
 耦合：程序间的依赖关系
        包括：
             类之间的依赖
             方法间的依赖
 解耦：
     降低程序间的依赖关系
 实际开发中，
     **应该做到，编译器不依赖，运行时才依赖。**
 解耦的思路：
     第一步，使用反射来创建对象，而避免使用new关键字
     第二步，通过读取配置文件来获取要创建的对象全限定类名

**判定耦合的关键是，编译期间是否产生了依赖，做到编译不依赖，运行才依赖。**

### 3.2 工厂模式解耦

采用工厂模式来构建需要重用组件。即Bean。

如何解耦？1. 写配置文件。2. 通过反射来创建对象。

先看耦合版本:

```java
ui:层
public class Client {
    public static void main(String[] args) {
        IAccountService as = new AccountServiceImpl();//耦合
//        IAccountService as = (IAccountService) BeanFactory.getBean("accountService");
        as.saveAccount();

    }
}
业务层：
public class AccountServiceImpl implements IAccountService {

    private IAccountDao accountDao = new AccountDaoImpl();  // 耦合
//    private IAccountDao accountDao = (IAccountDao) BeanFactory.getBean("accountDao");

    @Override
    public void saveAccount() {
        accountDao.saveAccount();
    }
}

持久层:
public class AccountDaoImpl implements IAccountDao {
    @Override
    public void saveAccount() {
        System.out.println("保存了账户");
    }
}
```

可以看到ui和业务层都用new依赖着相光的对象。

现在使用工厂来解耦:

1. 配置文件

   ```
   accountService=service.impl.AccountServiceImpl
   accountDao=dao.impl.AccountDaoImpl
   ```

2. 工厂类

   ```java
   
   /**
    * @author Raven
    * @version 1.0
    * @date 2020/3/21 9:25
    * 创建bean对象的工厂
    *
    * Bean: 在计算机英语中，有可重用组件的含义。
    * JavaBean：用java语言编写的可重用组件
    *          javaBean != 实体类
    *          javaBean > 实体类
    *
    * 创建我们的service和dao对象的。
    * 第一个，需要一个配置文件来配置我们的service和dao
    *      配置内容：唯一标识=全限定类名 key-value的形式
    * 第二个，通过读取配置文件中配置的内容，反射创建对象
    *
    *  我们的配置文件可以是xml也可以properties
    */
   public class BeanFactory {
       private static Properties props;
   
       // 使用静态代码块为properties对象赋值
       static{
           try{
               // 实例化对象
               props = new Properties();
               // 获取props文件的流对象
               InputStream in = BeanFactory.class.getClassLoader().getResourceAsStream("bean.properties");
               props.load(in);
   
           }catch(Exception e){
               throw new ExceptionInInitializerError("初始化properties失败");
           }
   
       }
   
       public static  Object getBean(String beanName) {
           Object bean = null;
           try{
               String beanPath = props.getProperty(beanName);
               bean = Class.forName(beanPath).getConstructor().newInstance();
           }catch(Exception e){
               e.printStackTrace();
           }
           return bean;
       }
   }
   ```

   3. 应用工厂解耦（看上面耦合代码的注释部分即可）

上面的代码虽然实现了解耦，但是每次get时都会新建一个对象，所以可以通过饿汉方式建立单例模式。

### 3.3 new对象是主动的

```
 private IAccountDao accountDao = new AccountDaoImpl();  // 耦合
```

![image-20200321103252864](https://i.loli.net/2020/03/21/qt7x1UCVGvHQN9E.png)

### 3.4 工厂方法是被动

```java
private IAccountDao accountDao = BeanFactory.getBean("accountDao");
```

![image-20200321103306042](https://i.loli.net/2020/03/21/7X9j4V3uOnR8QWN.png)

### 3.5 控制翻转

![image-20200321103528330](https://i.loli.net/2020/03/21/ZkAtaFVRDHhQWwp.png)

**明确ioc的作用： 削减计算机程序的耦合(解除我们代码中的依赖关系)。**

## 4. Srping的IOC解决程序的耦合

Spring包机构

![image-20200321104435481](https://i.loli.net/2020/03/21/nKTgVaGtEwyi6x4.png)

### 4.1 spring demo

创建一个bean.xml配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="accountService" class="service.impl.AccountServiceImpl">
        <!-- collaborators and configuration for this bean go here -->
    </bean>

    <bean id="accountDao" class="dao.impl.AccountDaoImpl">
        <!-- collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions go here -->

</beans>
```

改写client:

```java
 public static void main(String[] args) {
        // 1. 获取核心容器对象
        ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");
        // 2. 根据id获取Bean对象
        IAccountService as = ac.getBean("accountService",IAccountService.class);
        IAccountDao aDao = ac.getBean("accountDao",IAccountDao.class);
       System.out.println(as);
       System.out.println(aDao);
    }
```

### 4.2 spring中常用类

**三个常用ApplicationContext** 

 1.  ClassPathXmlApplicationContext, 加载雷鲁静下的配置文件，要求配置文件必须在类路径下(更常用）
 2.  FileSystemXmlApplicationContext， 加载磁盘任意路径下的配置文件
  3.  AnnotationConfigApplicationContext, 加载注解

**核心容器的两个接口引发的问题：**

- ApplicationContext: **单例对象使用 更多采用此接口**
      在构建核心容器时，创建对象采取的策略是是立即加载，只要一读取玩配置文件，马上加载配置文件中的对象
         ApplicationContext as = new ClassPathXmlApplicationContext("bean.xml");
- BeanFactory:  **多例对象使用**
      在构建核心容器时，创建对象采取的策略是采用延迟加载的方式.
       BeanFactory ac = new XmlBeanFactory(new ClassPathResource("bean.xml"));

### 4.3  spring中创建bean的三种方式

##### 1.普通方式

```xml
<!--
    1.1 在spring的配置文件中使用bean标签，配以id和class属性之后，且没有其它属性和标签时
    采用的是默认构造函数创建bean对象，此时如果类中没有默认构造函数，则对象无法创建
-->
    <bean id="accountService" class="service.impl.AccountServiceImpl"/>
```

##### 2.通过类成员函数

```xml
<!--
    1.2 使用普通工厂中的方法创建对象，使用某个类中的方法创建对象，并存入spring容器
-->
    <bean id="instanceFactory" class="factory.StaticFactory"/>
    <bean id="accountService"  factory-bean="instanceFactory" factory-method="getAccountService"/>
    <!-- more bean definitions go here -->
```

##### 3.通过类静态方法

```xml

<!--
    1.3 使用工厂中的静态方法创建对象，使用某个类中的静态方法创建对象，并存入spring
-->
    <bean id="accountService" class="factory.StaticFactory" factory-method="getAccountService"/>
```

### 4.4 bean对象的作用范围

bean对象的作用范围

通过scope属性，用于指定bean的作用范围

取值： 常用的就是单例的和多例

- ​    singleton 单例 默认
- ​    prototype 多例
- ​    request 作用于web应用的请求范围
- ​    session 作用于web应用的会话范围
- ​    global-session  作用于集群环境的会话范围（全局会话范围），当不是集群环境是，他就是session

![](https://i.loli.net/2020/03/21/kDNTjQAflOGZ5JR.png)

### 4.5  bean对象的生命周期

##### 1.单例对象

单例对象的生命周期和容器相同

##### 2.多例对象

出生：当使用对象，spring框架创建
活着：对象被使用着，就会一直活着
死亡：当对象长时间不用，且没有别的对象引用时，由Java的垃圾回收期回收

### 4.6 spring的依赖注入

```
  spring中的依赖注入
    dependency inject
    IOC的作用：
        降低程序间的耦合（依赖关系）
    依赖关系的管理：
        以后都交给spring来维护
     在当前类需要用到其它类的对象，由spring提供，只需要在配置文件中说明
     依赖关系的维护：
        就称之为依赖注入。
        
     依赖注入：
        能注入的数据：有三类
                1. 基本类型和String
                2. 其它bean类型（在配置文件中或者注解配置过的bean）
                3. 复杂类型/集合类型
         注入的方式：有三种
                1. 构造函数提供
                2. 使用set方法提供
                3. 使用注解提供
```

##### 1.构造函数的方式

```
构造函数注入 一般不采用
优势：
    在获取bean对象时，注入数据是必须的操作，否则对象无法创建成功。
弊端：
    改变了bean对象的实例化方式，使我们在创建对象时，如果用不到这些数据，也必须提供。
使用的标签：constructor-arg
标签出现的位置：bean标签的内部
标签的属性
    type: 用于指定要注入的数据的数据类型，该数据类型也是构造函数中某个或某些参数的类型
    index: 用于指定要注入的数据给构造函数中指定索引位置的参数赋值。索引的位置是从0开始
    name；用于指定给构造函数中指定名称的参数赋值。   常用！
    ===========以上三个用于指定给哪个构造函数中哪个参数赋值======================
    value: 用于提供基本类型和String类型的数据
    ref: 引用xml中的其它对象，用于指定其他的bean类型数据。
    它指的就是在spring的ioc核心容器中出现过的bean对象
```

```xml
  <bean id="accountService" class="service.impl.AccountServiceImpl">
        <constructor-arg name="name" value="test"></constructor-arg>
        <constructor-arg name="age" value="18"></constructor-arg>
        <constructor-arg name="birth" ref="now"></constructor-arg>
    </bean>
<!--
    配置一个日期对象
-->
    <bean id="now" class="java.util.Date">
        <constructor-arg name="year" value="2004"></constructor-arg>
        <constructor-arg name="month" value="12"></constructor-arg>
        <constructor-arg name="date" value="01"></constructor-arg>
    </bean>
```

```java
public AccountServiceImpl(String name, Integer age, Date birth){
    this.name = name;
    this.age = age;
    this.birth = birth;
}
```

##### 2.set方法

```xml
<!--
     set方法注入 更常用
        优势缺点与constructor相反
        优势：
            创建对象没有明确限制，可以直接使用构造函数
        弊端：
            如果某个成员必须有值，可能set方法无法保证一定注入
     使用property
     出现的位置：bean内部
-->
    <bean id="accountService2" class="service.impl.AccountServiceImpl2">
        <property name="name" value="raven"/>
        <property name="age" value="18"/>
        <property name="birth" ref="now"/>
    </bean>
```

```java
// 如果是经常变化的数据，并不适用与注入的方式
private String name;
private Integer age;
private Date birth;

public void setName(String name) {
    this.name = name;
}

public void setAge(Integer age) {
    this.age = age;
}

public void setBirth(Date birth) {
    this.birth = birth;
}
```

##### 3.复杂类型注入

```xml
<!--
     复杂类型的注入
     用于给List结构集合注入的标签
     list,array,set
     用于给Map结构注入的标签有：
     map,props
     只要结构相同，可以互换
-->
    <bean id="accountService3" class="service.impl.AccountServiceImpl3">
        <property name="myArray">
            <array>
                <value>AAA</value>
                <value>BBB</value>
                <value>CCC</value>
            </array>
        </property>
        <property name="myList">
            <list>
                <value>LAAA</value>
                <value>LBBB</value>
                <value>LCCC</value>
            </list>
        </property>
        <property name="mySet">
            <set>
                <value>SAAA</value>
                <value>SBBB</value>
                <value>SCCC</value>
            </set>
        </property>
        <property name="myMap">
            <map>
                <entry  key="KA" value="AAA"/>
                <entry  key="KB" value="BBB"/>
                <entry  key="KC" value="CCC"/>
            </map>
        </property>

        <property name="myProps">
            <props>
                <prop key="KA">A</prop>
                <prop key="KB">B</prop>
            </props>
        </property>
    </bean>
```

```xml
public class AccountServiceImpl3 implements IAccountService {

    private String[] myArray;
    private List<String> myList;
    private Set<String> mySet;
    private Map<String,String> myMap;
    private Properties myProps;

    public void setMyArray(String[] myArray) {
        this.myArray = myArray;
    }

    public void setMyList(List<String> myList) {
        this.myList = myList;
    }

    public void setMySet(Set<String> mySet) {
        this.mySet = mySet;
    }

    public void setMyMap(Map<String, String> myMap) {
        this.myMap = myMap;
    }

    public void setMyProps(Properties myProps) {
        this.myProps = myProps;
    }

    @Override
    public void saveAccount() {
        System.out.println(toString());
    }

    @Override
    public String toString() {
        return "AccountServiceImpl3{" +
                "myArray=" + Arrays.toString(myArray) +
                ", myList=" + myList +
                ", mySet=" + mySet +
                ", myMap=" + myMap +
                ", myProps=" + myProps +
                '}';
    }
}
```