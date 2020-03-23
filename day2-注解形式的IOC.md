[toc]

![image-20200321185006008](https://i.loli.net/2020/03/22/D7q1acweQvzVFLj.png)

## 1. 注解

1. 用于创建对象的的作用就和在XM配置文件中编写一个<bean>标签实现的功能是一样的
2. 用于注入作用就和在xm配置文件中的bean标签中写一个< property>标签的作用是一样的
3. 用于改变作用范围的他们的作用就利在bean标签中使用 scope属性实现的功能是一样的
4. 生命周他们的作用就和在bean标签中使用init- method和 destroy- methode的作用是一样的

### 1. 创建对象

xml方式：

```xml
<bean id="accountService" class="service.impl.AccountServiceImpl">
```

注解方式：

@**Component** 注解，创建对象
         属性：value，作为获取Bean的key。如果不写，默认为当前类名首字母小写

```
@Component
public class AccountServiceImpl implements IAccountService 
```

开启bean.xml的注解扫描.注意引入context命名空间。

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="dao.impl service.impl"/>
```

其余可用注解：

Controller：一般用在表现层
Service：一般用在业务层
Repository：一般用在持久层
以上三个注解他们的作用和属性与 Component是一模一样
他们三个是 spring框架为我们提供明确的三层使用的注解，使我们的三层对象更加清晰



### 2. 依赖注入

xml方式：

```xml
<bean id="accountService" class="service.impl.AccountServiceImpl">
    <constructor-arg name="name" value="test"></constructor-arg>
    <constructor-arg name="age" value="18"></constructor-arg>
    <constructor-arg name="birth" ref="now"></constructor-arg>
</bean>
```

**1. Autowired**
作用：自动按照类型注入。只要容器中有唯一的一个bean对象类型和要注入的变量类型匹配，就可以注入成功。

- 如果ioc容器中没有任何bean的类型和要注入的变量类型匹配，则报错
- 如果Ioc容器中有多个类型匹配时，根据变量名称进行注入

出现位置
	可以是变量上，也可以是方法上
细节：
	在使用注解注入时，set方法就不是必须的了。

![](https://i.loli.net/2020/03/22/pzhgGNdmlawUonv.png)

首先根据数据类型去spring的ioc容器中寻找object，当寻找到多个匹配时，根据变量名称进行匹配。

2.Qualifier，当按照数据类型已经无法区别要使用那个object时，可以再添加qualifier。注意qualifier不能单独使用（除在成员函数上）

```java
@Autowired
@Qualifier("accountDao2")
private IAccountDao accountDao;
```

3.Resource，可以单独使用，但是属性名变成了name.

```java
@Resource(name = "accountDao1")
private IAccountDao accountDao;
```

4.基本类型和 string类型的注入

@Value 用于注入基本类型和String类型。value：用于指定数据的值。它可以使用 spring中SpEL（也就是 spring的el表达式）

```java
@Value("raven")
private String name;
```

5.集合类型的注入只能通过MM来实现

### 3. 作用范围

xml方式:

scope="sington|propotype|xxx"

@Scope  属性value = singleton|prototype等

### 4. 生命周期

xml方式：

```xml
    <bean id="accountService" class="service.impl.AccountServiceImpl"
        init-method="" destroy-method=""
    >
```

@PreDestroy : 绑定销毁 

@PostConstruct : 绑定初始化

## 2. 使用注解去掉xml文件

要去掉xml文件，首先要找到一个和xml文件类型功能的文件。

这里可以通过新建Class+Annotation的方式来代替。

如新建一个SpringConfiguration.java, 并且使用Configuration注解。

```
Configuration
     作用：指定当前类是一个配置类
     细节：当配置类作为AnnotationConfigApplicationContext对象创建的参数时，该注解可以不写。
     通过ComponentScan扫描时必须谢Configuration注解。
```

```java
@Configuration
public class SpringConfiguration {

}
```

如何加入spring ioc容器支持注解的那些“自带类”，如service和dao等？

通过@ComponentScan注解即可。

```xml
ComponentScan
     作用：用于通过注解指定spring在创建容器时要扫描的包
     属性：value或basePackages的作用是一样的，都是用于指定创建容器时要扫描的包
            使用此注解就等同于在xml中配置了
                <context:component-scan base-package="dao.impl service.impl"/>
```

```java
@ComponentScan({"dao.impl","service.impl","config"})
public class SpringConfiguration {

}
```

我们还可以建立配置父子依赖关系，使用Import注解即可。如我们还有一个JDBCConfig.java配置：

```java
public class JDBCConfig{}
@Configuration
@Import({JDBCConfig.class})
@ComponentScan({"dao.impl","service.impl","config"})
public class SpringConfiguration {

}

```

如何新建jar包中所需要的类？通过Bean注解即可。

```
Bean
     作用：用于把当前方法的返回值作为bean对象存入spring的ioc容器中
     name: 用于指定bean的id，当不写时，默认值就是当前方法的名称
     细节：
         当使用注解配置方法时，如果方法有参数，spring框架会去容器中朝朝有没有可用的bean对象
         查找的方法和autowired的作用是一样的。
```

```java
/**
 *
 * @param dataSource
 * @return
 */
@Bean("runner")
@Scope("prototype")
// 可以使用Qualifier注解可以指定要使用dataSource（如果存在多个dataSource）
//  public QueryRunner createQueryRunner(@Qualifier("dataSource1") DataSource dataSource){
public QueryRunner createQueryRunner(DataSource dataSource){
    return new QueryRunner(dataSource);
}

 @Bean("dataSource")	// 如果没有这个Bean，上面createQueryRunner中的DataSource会报错，dataSource会向spring ioc容器中查找相应的object。
    public DataSource createDataSource(){
        ComboPooledDataSource dataSource =  new ComboPooledDataSource();
        try{
            dataSource.setDriverClass("com.mysql.cj.jdbc.Driver");
            dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/spring?serverTimezone=UTC");
            dataSource.setUser("root");
            dataSource.setPassword("0512");
        }catch(Exception e){
            throw new RuntimeException(e);
        }
        return dataSource;
    }
```

上面的代码还有些问题，我们将jdbc的左右信息都写死了。需要改进。把jdbc的相关信息都抽取出来写成properties文件L

```
jdbc.driver=com.mysql.cj.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/spring?serverTimezone=UTC
jdbc.user=root
jdbc.password=0512
```

然后使用@Value注解，使用spring的el表达式取得value：

```java
/**
 * jdbc相关配置
 */
@Value("${jdbc.driver}")
private String driverName ;
@Value("${jdbc.url}")
private String jdbcUrl;
@Value("${jdbc.user}")
private String userName;
@Value("${jdbc.password}")
private String password;
```

还需要注明从哪个文件中取，这里就要用到PropertySource注解。

PropertySource
     作用，用于指定properties文件的位置
     属性：
         values： 指定文件的名称和路径
                 关键字：classpath，表示类路径下

```java
@Configuration
@PropertySource("classpath:jdbc.properties")
public class JDBCConfig {
```

### 注解使用原则

由于注解并不能减少工作量，所使用注解的原则：哪个更方便使用哪个。

自己写的service、dao等可以使用注解。但是已经封装好的jar包，使用xml更方便。

## 3. spring整合junit

1. 应用程序的入口

   ​	main方法

2. junit单元测试中，没有main方法也能执行

   ​	junit集成了一个main方法，该方法就会判断当前测试类中哪些方法有@Test注解

   ​	junit就会让让有Test注解的方法执行

3. junit不会管是否采用spring框架

   ​	在执行测试方法时，junit根本不知道我们是否使用了spring框架。所以也就不会为我们读取配置文件，创建spring核心容器。

4. 由以上三点可知：

   当测试方法执行时，没有ioc容器，就算写了autowired注解，也无法注入依赖。

总体来说想办法替换spring的main方法。

**Spring整合junit**
 1、 导入spring整合junit的jar包

 2、 使用junit提供的注解，把原有的main方法替换了，替换成spring提供的
    	 @Runwith

 3、 告知spring的运行器，spring和ioc创建是基于xml还是注解的，并且说明位置
    	 @ContextConfiguration
     		location 指定xml文件的位置，加上classpath关键字，表示在类
  		   classes: 指定注解类所在的位置
*当使用spring5.x版本，要求junit的jar包必须是4.12及以上*

spring有相关的整合包：

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-test</artifactId>
    <version>5.2.4.RELEASE</version>
</dependency>
```

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = SpringConfiguration.class)
public class Client {
    
   //===================或者
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath:bean.xml"})
    public class Client {
```

接下来就可以直接注入了。

```java
@Autowired
IAccountService as;
```