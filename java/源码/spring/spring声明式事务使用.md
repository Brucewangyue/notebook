## 什么是事务

> 把一组业务当成一个业务来做；要么都成功，要么都失败，保证业务操作完整性的一种数据库机制。

## Spring JdbcTemplate

在spring中为了**更加方便的操作JDBC**，在JDBC的基础之上定义了一个抽象层，此设计的目的是为不同类型的**JDBC操作**提供**模板方法**，每个模板方法都能控制整个过程，并允许覆盖过程中的特定任务，通过这种方式，可以尽可能保留灵活性，将数据库存取的工作量降到最低。

**1、配置并测试数据源**

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>5.2.6.RELEASE</version>
        <scope>compile</scope>
    </dependency>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid</artifactId>
        <version>1.1.21</version>
    </dependency>

    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>5.1.47</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aop</artifactId>
        <version>5.2.6.RELEASE</version>
        <scope>compile</scope>
    </dependency>
    <!--@Aspectj 注解、表达式匹配-->
    <dependency>
        <groupId>org.aspectj</groupId>
        <artifactId>aspectjweaver</artifactId>
        <version>1.9.5</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aspects</artifactId>
        <version>5.2.6.RELEASE</version>
        <scope>compile</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-orm</artifactId>
        <version>5.2.6.RELEASE</version>
        <scope>compile</scope>
    </dependency>
</dependencies>
```

dbconfig.properties

```properties
jdbc.username=root123
password=123456
url=jdbc:mysql://localhost:3306/demo
driverClassName=com.mysql.jdbc.Driver
```

jdbcTemplate.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:context="http://www.springframework.org/schema/context"
      xmlns:aop="http://www.springframework.org/schema/aop"
      xsi:schemaLocation="http://www.springframework.org/schema/beans
      http://www.springframework.org/schema/beans/spring-beans.xsd
      http://www.springframework.org/schema/context
      http://www.springframework.org/schema/context/spring-context.xsd
      http://www.springframework.org/schema/aop
      https://www.springframework.org/schema/aop/spring-aop.xsd
">
   <context:property-placeholder location="classpath:dbconfig.properties"></context:property-placeholder>
   <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
       <property name="username" value="${jdbc.username}"></property>
       <property name="password" value="${jdbc.password}"></property>
       <property name="url" value="${jdbc.url}"></property>
       <property name="driverClassName" value="${jdbc.driverClassName}"></property>
   </bean>
   <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
       <constructor-arg name="dataSource" ref="dataSource"></constructor-arg>
   </bean>
    <!--具备具名函数的jdbcTemplate-->
    <bean id="namedParameterJdbcTemplate" class="org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate">
       <constructor-arg name="dataSource" ref="dataSource"></constructor-arg>
   </bean>
    
    <context:component-scan base-package="org.my"></context:component-scan>
</beans>
```

MyTest.java

```java
import com.alibaba.druid.pool.DruidDataSource;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import java.sql.SQLException;

public class MyTest {
    static ApplicationContext context;
    static JdbcTemplate jdbcTemplate;
    static NamedParameterJdbcTemplate namedJdbcTemplate;
    
   public static void main(String[] args) throws SQLException {
       context = new ClassPathXmlApplicationContext("jdbcTemplate.xml");
       // 测试连接
       DruidDataSource dataSource = context.getBean("dataSource", DruidDataSource.class);
       System.out.println(dataSource);
       System.out.println(dataSource.getConnection());
       
       jdbcTemplate = context.getBean("jdbcTemplate", JdbcTemplate.class);
       namedJdbcTemplate = context.getBean("namedParameterJdbcTemplate", NamedParameterJdbcTemplate.class);
       
       insert();
       
  }
    
    public static void insert(){
       String sql = "insert into emp(empno,ename) values(?,?)";
       int result = jdbcTemplate.update(sql, 1111, "zhangsan");
       System.out.println(result);
    }
    
    public static void batchInsert(){
       List<Object[]> list = new ArrayList<Object[]>();
       list.add(new Object[]{1,"zhangsan1"});
       list.add(new Object[]{2,"zhangsan2"});
       list.add(new Object[]{3,"zhangsan3"});
       int[] result = jdbcTemplate.batchUpdate(sql, list);
       for (int i : result) {
           System.out.println(i);
      }
    }
    
    public static void queryForObject(){
       String sql = "select * from emp where empno = ?";
       Emp emp = jdbcTemplate.queryForObject(sql, new BeanPropertyRowMapper<>(Emp.class), 7369);
       System.out.println(emp);
    }
    
    public static void queryList(){
        String sql = "select * from emp where sal > ?";
        List<Emp> query = jdbcTemplate.query(sql, new BeanPropertyRowMapper<>(Emp.class), 1500);
        for (Emp emp : query) {
            System.out.println(emp);
        }
    }
    
     // 返回单个数
     public static void querySingle(){
        String sql = "select max(sal) from emp";
        Double aDouble = jdbcTemplate.queryForObject(sql, Double.class);
        System.out.println(aDouble);
     }
    
    public static void insertByNamed(){
        String sql = "insert into emp(empno,ename) values(:empno,:ename)";
        Map<String,Object> map = new HashMap<>();
        map.put("empno",2222);
        map.put("ename","sili");
        int update = namedJdbcTemplate.update(sql, map);
        System.out.println(update);
    }
    
    
}
```

EmpDao

```java
import cn.tulingxueyuan.bean.Emp;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;

public class EmpDao {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    public void save(Emp emp){
        String sql = "insert into emp(empno,ename) values(?,?)";
        int update = jdbcTemplate.update(sql, emp.getEmpno(), emp.getEname());
        System.out.println(update);
    }
}
```



## 声明式事务 

在事务控制方面，主要有两个分类：

1. **编程式事务**：在代码中直接加入处理事务的逻辑，可能需要在代码中显式调用beginTransaction()、commit()、rollback()等事务管理相关的方法
2. **声明式事务**：在方法的外部添加注解或者直接在配置文件中定义，将事务管理代码从业务方法中分离出来，以声明的方式来实现事务管理。spring的AOP恰好可以完成此功能：事务管理代码的固定模式作为一种横切关注点，通过AOP方法模块化，进而实现声明式事务

### 简单配置

Spring从不同的事务管理API中抽象出了一整套事务管理机制，让事务管理代码从特定的事务技术中独立出来。开发人员通过配置的方式进行事务管理，而不必了解其底层是如何实现的。

Spring的核心事务管理抽象是PlatformTransactionManager。它为事务管理封装了一组独立于技术的方法。无论使用Spring的哪种事务管理策略(编程式或声明式)，事务管理器都是必须的。

事务管理器可以以普通的bean的形式声明在Spring IOC容器中。下图是spring提供的事务管理器

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/aop
       https://www.springframework.org/schema/aop/spring-aop.xsd
       http://www.springframework.org/schema/tx
       https://www.springframework.org/schema/tx/spring-tx.xsd
"> 
	<!--事务控制-->
    <!--配置事务管理器的bean-->
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"></property>
    </bean>
    <!--开启基于注解的事务控制模式，依赖tx名称空间-->
    <tx:annotation-driven transaction-manager="transactionManager"></tx:annotation-driven>
</beans>
```

**@Transactional注解应该写在哪：**

@Transactional   可以标记在类上面（当前类所有的方法都运用上了事务）

@Transactional   标记在方法则只是当前方法运用事务

也可以类和方法上面同时都存在， 如果类和方法都存在@Transactional会以方法的为准。如果方法上面没有@Transactional会以类上面的为准

建议：@Transactional写在方法上面，控制粒度更细，   建议@Transactional写在业务逻辑层上，因为只有业务逻辑层才会有嵌套调用的情况。

### 事务配置的属性

- isolation：设置事务的隔离级别

- propagation：事务的传播行为

- noRollbackFor：那些异常事务可以不回滚

- noRollbackForClassName：填写的参数是全类名

- rollbackFor：哪些异常事务需要回滚

- rollbackForClassName：填写的参数是全类名

- readOnly：设置事务是否为只读事务	

- timeout：事务超出指定执行时长后自动终止并回滚,单位是秒

### 设置事务的隔离级别

isolation

用来解决并发事务所产生一些问题：对同一个数据（变量、对象）进行读写操作才会产生并发问题

并发会产生什么问题？

1. **脏读**：一个事务，读取了另一个事务中没有提交的数据，会在本事务中产生的数据不一致的问题

   | **事务1**  begin                                     | **事务2**  begin                                     |
   | ---------------------------------------------------- | ---------------------------------------------------- |
   |                                                      | update t_user set balance=800where id=1;#balance=800 |
   | select * from  t_user where id=1commit; #balance=800 |                                                      |
   |                                                      | rollback;  #回滚#balance=1000                        |

   **解决方式：读已提交 @Transactional(isolation = Isolation.READ_COMMITTED)**

2. **不可重复读**：一个事务中，多次读取相同的数据，但是读取的结果不一样，会在本事务中产生数据不一致的问题

   | **事务1**  begin                              | **事务2**  begin                                             |
   | --------------------------------------------- | ------------------------------------------------------------ |
   | select * from  t_user where id=1#balance=1000 |                                                              |
   |                                               | update t_user set balance=800where id=1;commit; #balance=800 |
   | select * from t_user where id=1#balance=800   |                                                              |
   | commit;                                       |                                                              |

   **解决方式：可重复读@Transactional(isolation = Isolation.REPEATABLE_READ)**

3. **幻读**：一个事务中，多次对数据进行**整表或者范围**读取（统计），但是结果不一样， 会在本事务中产生数据不一致的问题

   | **事务1**  begin                                         | **事务2**  begin                                             |
   | -------------------------------------------------------- | ------------------------------------------------------------ |
   | select sum(balance) from  t_user where id=1#balance=3000 |                                                              |
   |                                                          | INSERT INTO  t_userVALUES	(		'4',		'赵六',		'123456784',		'1000'	);commit; |
   | select sum(balance) from  t_user where id=1#balance=4000 |                                                              |
   | commit;                                                  |                                                              |

   **解决方式：串行化 @Transactional(isolation = Isolation.SERIALIZABLE)**

   串行化会出现，行锁（条件只有一条数据）、间隙锁（范围查询）、表锁（全表查询）

4. **很多人容易搞混不可重复读和幻读，确实这两者有些相似：**

    对于前者,  只需要锁行

    对于后者,  需要锁表 



### 事务的传播行为

propagation

事务的传播特性指的是当一个事务方法被另一个事务方法调用时，这个事务方法应该如何进行？

| **事务传播行为类型** | **外部不存在事务** | **外部存在事务**                                             | **使用方式**                                                 |
| -------------------- | ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| REQUIRED（默认）     | 开启新的事务       | 融合到外部事务中                                             | @Transactional(propagation = Propagation.REQUIRED)适用增删改查 |
| SUPPORTS             | 不开启新的事务     | 融合到外部事务中                                             | @Transactional(propagation = Propagation.SUPPORTS)适用查询   |
| REQUIRES_NEW         | 开启新的事务       | 不用外部事务，创建新的事务                                   | @Transactional(propagation = Propagation.REQUIRES_NEW)适用内部事务和外部事务不存在业务关联情况，如日志 |
| NOT_SUPPORTED        | 不开启新的事务     | 不用外部事务                                                 | @Transactional(propagation = Propagation.NOT_SUPPORTED)不常用 |
| NEVER                | 不开启新的事务     | 抛出异常                                                     | @Transactional(propagation = Propagation.NEVER )不常用       |
| MANDATORY            | 抛出异常           | 融合到外部事务中                                             | @Transactional(propagation = Propagation.MANDATORY)不常用    |
| NESTED               | 开启新的事务       | 融合到外部事务中,SavePoint机制，外层影响内层， 内层不会影响外层 | @Transactional(propagation = Propagation.NESTED)不常用       |



### 异常属性

设置 当前事务出现的那些异常就进行回滚或者提交。

默认对于RuntimeException 及其子类 采用的是回滚的策略。

默认对于Exception 及其子类 采用的是提交的策略。 

1. 设置哪些异常不回滚(noRollbackFor)
2. 设置哪些异常回滚（rollbackFor ）

```java
@Transactional(timeout = 3,rollbackFor = {FileNotFoundException.class})
```





### 只读事务

就是业务代码端的 可重复读

connection.setReadOnly(true)    通知数据库，当前数据库操作是只读，数据库就会对当前只读做相应优化

开启只读事务模式之后，事务期间任何insert、update或delete都是不允许的

那么问题来了 ，这个和不开事务有什么区别？

区别还是有的，如果使用的是默认的事务隔离级别“可重复读(rr：reaptable read)”，那么在只读事务开启后，如果重复读取某表数据，那么结果是一致的，并不会受到事务外其他数据的提交影响



### 超时属性

timeout

当前事务访问数据时，有可能访问的数据被别的数据进行加锁的处理，那么此时事务就必须等待，如果等待时间过长给用户造成的体验感差



### 在实战中事务的使用方式

1. 如果当前业务方法是一组 增、改、删  可以这样设置事务

   @Transactional

2. 如果当前业务方法是一组 查询  可以这样设置事务

   @Transactionl(readOnly=true)

3. 如果当前业务方法是单个 查询  可以这样设置事务

   @Transactionl(propagation=propagation.SUPPORTS ,readOnly=true)



## 基于xml的事务配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:context="http://www.springframework.org/schema/context"
      xmlns:aop="http://www.springframework.org/schema/aop"
      xmlns:tx="http://www.springframework.org/schema/tx"
      xsi:schemaLocation="http://www.springframework.org/schema/beans
      http://www.springframework.org/schema/beans/spring-beans.xsd
      http://www.springframework.org/schema/context
      http://www.springframework.org/schema/context/spring-context.xsd
      http://www.springframework.org/schema/aop
      https://www.springframework.org/schema/aop/spring-aop.xsd
      http://www.springframework.org/schema/tx
      https://www.springframework.org/schema/tx/spring-tx.xsd
">
   <context:component-scan base-package="cn"></context:component-scan>
   <context:property-placeholder location="classpath:dbconfig.properties"></context:property-placeholder>
   <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
       <property name="username" value="${jdbc.username}"></property>
       <property name="password" value="${jdbc.password}"></property>
       <property name="url" value="${jdbc.url}"></property>
       <property name="driverClassName" value="${jdbc.driverClassName}"></property>
   </bean>
   <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
       <constructor-arg name="dataSource" ref="dataSource"></constructor-arg>
   </bean>
   <!--事务控制-->
   <!--配置事务管理器的bean-->
   <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
       <property name="dataSource" ref="dataSource"></property>
   </bean>
   <!--
   基于xml配置的事务：依赖tx名称空间和aop名称空间
       1、spring中提供事务管理器（切面），配置这个事务管理器
       2、配置出事务方法
       3、告诉spring哪些方法是事务方法（事务切面按照我们的切入点表达式去切入事务方法）
   -->
   <bean id="bookService" class="cn.service.BookService"></bean>
   <aop:config>
       <aop:pointcut id="txPoint" expression="execution(* cn.service.*.*(..))"/>
       <!--事务建议：advice-ref:指向事务管理器的配置-->
       <aop:advisor advice-ref="myAdvice" pointcut-ref="txPoint"></aop:advisor>
   </aop:config>
   <tx:advice id="myAdvice" transaction-manager="transactionManager">
       <!--事务属性-->
       <tx:attributes>
           <!--指明哪些方法是事务方法-->
           <tx:method name="*"/>
           <tx:method name="checkout" propagation="REQUIRED"/>
           <tx:method name="get*" read-only="true"/>
       </tx:attributes>
   </tx:advice>
</beans>
```







