# 1、Spring IOC 应用

## 1.1、Spring IOC 基础

### 1.1.1、BeanFactroy与ApplicationContext区别

`BeanFactory` 是Spring 框架中 IoC 容器的顶层接口，它只是用来定义一些基础功能，定义一些基础规范，而 `ApplicationContext` 是它的一个子接口，所以 `ApplicationContext` 是具备 `BeanFacroty` 提供的全部功能的。

通常，我们称 `BeanFactory` 为 Spring IOC 的基础容器，`ApplicationContext` 是容器的高级接口，比 `BeanFactory` 要拥有更多的功能，比如说国际化支持和资源访问等等。  

![image-20200714162350101](04-Spring%20IOC%20%E5%BA%94%E7%94%A8/image-20200714162350101.png)

#### 1.1.1.1、启动IOC容器方式

- Java 环境

  - ClassPathXmlApplicationContext：从类的根路径下加载配置文件 :thumbsup:

  ```java
  ApplicationContext ctx = new ClassPathXmlApplication("beans.xml");
  ```

  - FileSystemXmlApplicationContext：从磁盘路径下加载配置文件

  ```java
  ApplicationContext ctx = new FileSystemXmlApplicationContext("c:\beans.xml");
  ```

  - AnnotationConfigApplicationContext：纯注解模式下启动容器

  ```java
  ApplicationContext ctx = new AnnotationConfigApplicationContext(SpringConfig.class);
  ```

  >  SpringConfig是一个基于注解实现的Spirng配置类

- Web 环境

  - 从XML启动容器

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                        http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
           version="3.1">
      <!--配置文件位置-->
      <context-param>
          <param-name>contextConfigLocation</param-name>
          <param-value>classpath*:spring/applicationContext-*.xml</param-value>
      </context-param>
      <!-- 使用监听器启动IOC容器-->
      <listener>
   		<listener-class>
              org.springframework.web.context.ContextLoaderListener
          </listener-class>
      </listener>
  </web-app>
  ```
  - 从配置类启动容器
  
  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
    <web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                          http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
             version="3.1">
        <!--告知监听器使用注解方式启动IOC容器-->
        <context-param>
            <param-name>contextClass</param-name>
            <param-value>
               org.springframework.web.context.support.AnnotationConfigWebApplicationContext
            </param-value>
        </context-param>
    	<!--配置启动类全限定名称-->
        <context-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>com.github.config.SpringConfig</param-value>
        </context-param>
        <!-- 使用监听器启动IOC容器-->
        <listener>
     		<listener-class>
                org.springframework.web.context.ContextLoaderListener
            </listener-class>
        </listener>
    </web-app>
  ```

### 1.1.2、纯XML方式使用IOC容器

#### 1.1.2.1、XML 文件头

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans.xsd">
</beans>
```

#### 1.1.2.2、实例化Bean的三种方式

- 方式一：使用无参构造函数

  - java

  ```java
  public class Person{
      public Person{
          System.out.println("Person 被实例化...");
      }
  }
  ```

  - xml 配置

  ```xml
  <bean id="person" class="com.github.bean.Person"/>
  ```

  > 默认情况下将会通过反射调用无参的构造函数进行实例化，如果类中没有无参的构造函数将会报错。

- 方式二：使用静态工厂方法

  - java

  ```java
  public class PersonFactory{
      public static Person createPerson(){
          return new Person();
      }
  }
  ```

  - xml 配置

  ```xml
  <bean id = "person" class = "com.github.factory.PersonFacroty" 
        factory-method = "createPerson"/>
  ```

  >  实际开发过程中，使用的对象有些时候并不是直接通过构造函数就可以创建出来的，它可能在创建过程中会做很多额外的操作。此时会提供一个创建对象的方法，恰好这个方法是 static 修饰的，这就是静态工厂方法。

- 方式三：使用实例化工厂方法
  - java

  ```java
  public class PersonFactory{
    public Person createPerson(){
          return new Person();
      }
  }
  ```

  - xml 配置

  ```xml
  <bean id = "personFacroty" class = "com.github.factory.PersonFacroty"/>
  <bean id = "person" factory-bean="personFacroty" factory-method = "createPerson"/>
  ```

  > 此种方式需要配置两个bean。第一个bean使用的类构造器实例化一个工厂类，第二个bean中的id是实例化对象的名称，factory-bean对应的被实例化的工厂类的对象名称，也就是第一个bean的id，factory-method是非静态工厂方法。使用此方法实例化对象时，需要先使用类构造器实例化一个工厂类。
  >

#### 1.1.2.3、Bean的作用域及生命周期

##### 1.1.2.3.1、作用域的改变

在 Spring 框架管理 Bean 对象的创建时，Bean 对象默认都是单例的，但是它支持配置的方式改变作用域范围。

| Scope       | Description                                                  |
| ----------- | ------------------------------------------------------------ |
| singleton   | (Default) Scopes a single bean definition to a single object instance for each Spring IoC container. |
| prototype   | Scopes a single bean definition to any number of object instances. |
| request     | Scopes a single bean definition to the lifecycle of a single HTTP request. That is, each HTTP request has its own instance of a bean created off the back of a single bean definition. Only valid in the context of a web-aware Spring .`ApplicationContext` |
| sessipn     | Scopes a single bean definition to the lifecycle of an HTTP . Only valid in the context of a web-aware Spring .`Session` `ApplicationContext` |
| application | Scopes a single bean definition to the lifecycle of a . Only valid in the context of a web-aware Spring .`ServletContext` `ApplicationContext` |
| webscoket   | Scopes a single bean definition to the lifecycle of a . Only valid in the context of a web-aware Spring .`WebSocket` `ApplicationContext` |

参考：[Spring作用域说明表](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#beans-factory-scopes)

实际开发中，用的最多的就是 `singleton`（单例模式）和 `Prototype` （原型模式）。配置方式参考如下：

```xml
<bean id="person" class="com.github.bean.Person" scope="prototype"/>
```

##### 1.1.2.3.2、不同作用范围的生命周期

- 单例模式：`singleton`

  - 对象出生：当创建容器时，对象就被创建了
  - 对象活着：只要容器在，对象一直活着
  - 对象死亡：当销毁容器时，对象就被销毁了

  > 单例模式的 Bean 对象的生命周期与容器相同

- 原型模式：`Prototype`
  - 对象出生：当使用对象时，创建新的对象实例
  - 对象活着：只要对象在使用中，就一直活着
  - 对象死亡：当对象长时间不用时，被 Java 的垃圾回收器回收了

  > 多例模式的 Bean 对象，Spring 框架只负责创建，不负责销毁

#### 1.1.2.4、Bean标签

在基于XML的IoC配置中，Bean 标签是最基础的标签。它标识了IoC容器中的一个对象。换句话说，如果一个对象想让spring管理，在XML的配置中都需要使用此标签配置，Bean标签的属性如下：

| 属性名         | 说明                                                         |
| :------------- | :----------------------------------------------------------- |
| id             | 用于给bean提供一个唯一标识。在一个配置文件内部，标识必须唯一 |
| class          | 用于指定创建Bean对象的全限定类名                             |
| name           | 用于给Bean提供一个或者多个名称，多个名称用空格分隔           |
| factory-bean   | 用于指定创建当前Bean对象的工厂Bean的唯一标识，当指定了此属性之后，class属性失效 |
| factory-method | 用于指定创建当前Bean对象的工厂方法，如果配和factory-bean属性。则class属性失效。如配和class属性使用，则方法必须是static的 |
| scope          | 用于指定Bean对象的作用范围。通常情况下就是singleton。当要用到多例模式时，可以配置为prototype |
| init-method    | 用于指定Bean对象的初始化方法，此方法会在Bean对象装配后调用，必须是一个无参方法 |
| destory-method | 用于指定Bean对象的销毁方法，此方法会在Bean对象销毁前执行，它只能为scope是singleton是起作用 |

#### 1.1.2.5、DI依赖注入的XML配置方式

- 依赖注入的分类
  - 按照注入方式分类
    - 构造函数注入：利用带参的构造函数实现对类成员的属性赋值
    - set方式注入：利用类成员的set方法实现数据的注入
  - 按照注入数据类型分类
    - 基本类型和String：注入的数据类型是基本类型或者是字符串类型的数据
    - 其他Bean类型：注入的数据类型是对象类型
    - 复杂类型（集合类型）：注入的数据类型是Array、List、Set、Map、Properties中的一种类型

##### 1.1.2.5.1、构造函数注入

- java

```java
public class User {
    private int uid;
    private String userName;
    private int age;
    public User(Integer uid, String userName) {
        super();
        this.uid = uid;
        this.userName = userName;
    }
    public User(String userName, Integer age) {
        super();
        this.userName = userName;
        this.age = age;
    }
    // 省略get/set方法
}
```

- xml配置

```xml
<!-- 方式一：类型type 和  索引 index -->
<bean id="user" class="com.github.bean.User" >
    <constructor-arg index="0" type="java.lang.String" value="1"/>
    <constructor-arg index="1" type="java.lang.Integer" value="2"/>
</bean>
<!-- 方式二：使用名称name -->
<bean id="user" class="com.github.bean.User" >
   <constructor-arg name="username" value="jack"/>
	<constructor-arg name="age" value="18"/>
</bean>
```

使用构造函数注入涉及到的标签为 `constructor-arg`，该标签有如下属性：

1. **index：**参数的索引号，从0开始 。如果只有索引，匹配到了多个构造方法时，默认使用第一个
2. **type：**确定参数类型
3. **value：**设置基本类型数据以及String类型数据
4. **name：**参数的名称
5. **ref：**引用数据，一般是另一个bean id值

##### 1.1.2.5.2、set方式注入

- java

```java
public class Address {
    private String id;
    private String name;
	// 省略get/set 方法
}
```

```java
public class City {
    private Address address;
    private String cityName;
    // 省略get/set 方法
}
```

- xml 配置

```xml
<bean id="address" class="com.github.bean.Address">
    <property name="id" value="123"/>
    <property name="name" value="新区"/>
</bean>
```

```xml
<bean id="city" class="com.github.bean.City">
    <property name="address" ref="address"/>
    <property name="cityName" value="无锡"/>
</bean>
```

在使用set方式注入时，需要使用到 `property` 标签，该标签有如下属性：

1. **name：**指定Bean 对象的属性名称
2. **value：**设置基本类型数据以及String类型数据
3. **ref：**引用数据，一般是另一个bean id值

> property 可以省略利用p标签实现

##### 1.1.2.5.3、复杂数据类型注入

复杂数据类型指的是集合数据类型，集合数据类型分为两类。一类是 `List` （数组结构）另外一类是 `Map` 结构（键值对）。

- java

```java
public class CollectionData {
    private String[] arrayData;
    private List<String> listData;
    private Set<String> setData;
    private Map<String, String> mapData;
    private Properties propsData;
    // 省略get/set 方法
}
```

- xml 配置

```xml
<bean id="collData" class="com.github.bean.CollectionData">
    <property name="arrayData">
        <array>
            <value>hello</value>
            <value>world</value>
        </array>
    </property>
    <property name="listData">
        <list>
            <value>A</value>
            <value>B</value>
        </list>
    </property>
    <property name="setData">
        <set>
            <value>C</value>
            <value>D</value>
        </set>
    </property>
    <property name="mapData">
        <map>
            <entry key="ZhangSan" value="E"/>
            <entry key="LiSi" value="F"/>
        </map>
    </property>
    <property name="propsData">
        <props>
            <prop key="WangWu">G</prop>
            <prop key="ZhaoLiu">H</prop>
        </props>
    </property>
</bean>
```

在 `List` 结构的集合数据注入时，`array` `list` `set` 这三个标签通用，另外注值的 `value`标签内部可以直接写值，也可以使用 `bean`标签配置的一个对象，或者用 `ref` 标签引用一个已经配好的 Bean的唯一标识。

在 `Map` 结构的集合数据注入时，`map` 标签使用了 `entry` 子标签实现数据注入，`entry` 标签可以使用 `key` 和 `value` 属性指定存入`map` 中的数据。使用 `value-ref` 属性指定已经配置好的 `bean` 的引用。同时 `entry` 标签中也可以使用 `ref` 标签，但是不能使用 `bean` 标签。而 `property` 标签中不能使用 `ref` 或者 `bean` 标签引用对象。 

### 1.1.3、XML与注解相结合方式使用IOC容器

1. 实际开发中，纯XML模式使用已经很少了
2. 引入注解功能，不需要引入额外的jar
3. XML+注解结合模式，XML文件依然存在，所以，Spring IOC 容器的启动任然从加载XML开始
4. 哪些bean的定义卸载XML中？哪些bean的定义使用注解？
   1. 第三方jar中的bean定义在XML中，例如druid数据库连接池，mybatis的 数据库会话等等
   2. 自己开发的bean 使用注解

#### 1.1.3.1、XML标签与注解对应表

| XML形式            | 注解形式                                                     |
| ------------------ | ------------------------------------------------------------ |
| bean 标签          | `@Component("accountDao")` ，注解加在类上，默认bean的id 为类的类名首字母小写。如果需要特殊指定，直接配置注解的属性值。另外，针对分层代码开发提供了`@Componment`的三种别名`@Controller` 、`@Service`、`@Repository`分别用于控制层类、服务层类、持久层类的bean定义，这四个注解的用法完全一样，只是为了更清晰的区分而已。 |
| scope属性          | `@Scope("prototype")`，默认单例，注解在类上                  |
| init-method属性    | `@Poststruct`，注解在方法上，该方法就是初始化后调用的方法    |
| destory-method属性 | `@PreDestory`，注解在方法上，该方法就是销毁前调用的方法      |

#### 1.1.3.2、DI 依赖注入的注解方式

##### 1.1.3.2.1、@Autowired

`@Autowired` 由Spring 提供的注解，需要导入`org.springframework.beans.factory.annotation.Autowired`

`@Autowired` 采用按照类型注入的策略

```java
public class UserService{
    @Autowired
    private UserDao userDao;
}
```

如上代码所示，这样装配会去Spring 容器中找到类型为 UserDao的类，然后将其注入进来。这样会产生一个问题，当一个类型有多个Bean值的时候，会造成无法选择具体注入哪一个的情况，这个时候需要我们配和`@Qualifier`使用。

`@Qualifier`告诉Spring 具体装配哪个对象。

```java
public class UserService{
    @Autowired
    @Qualifier("userDao")
    private UserDao userDao;
}
```

这个时候可以通过类型以及名称定位到具体的Bean，然后进行装配。

##### 1.1.3.2.2、@Resource

`@Resource` 由J2EE 提供，需要导入包`javax.annotation.Resource`

`@Resource` 默认按照名称进行自动注入

```java
public class UserService{
    @Resource
    private UserDao userDao;
    
    @Resource(name="personDao")
    private PersonDao personDao;
    
    @Resource(type="StudentDao")
    private StudentDao studentDao;
    
    @Resource(name="teacherDao",type="TeacherDao")
    private TeacherDao teacherDao;
}
```

1. 如果同时指定了 name 和 type，则从Spring 上下文找到唯一匹配的bean进行装配，找不到则抛出异常
2. 如果指定了 name，则从上下文中查找名称（id）匹配的 bean进行装配，找不到则抛出异常
3. 如果指定了 type，则从上下文中找到类似匹配的唯一 bean进行装配，找不到或是找到多个，都会抛出异常
4. 如果既没有指定 name，又没有指定 type，则自动按照 byName 方式进行装配

> @Resource 在JDK11 中已经被移除，如果要使用，需要单独引入jar包

### 1.1.4、纯注解方式使用IOC容器

| 注解            | 对应xml中的内容                                              |
| --------------- | ------------------------------------------------------------ |
| @Configuration  | 相当于 applicationConetxt.xml 文件，代表当前类是一个配置类   |
| @ComponentScan  | 相当于 `<context:component-scan>`，组件扫描器，将扫描到组件存入IOC容器 |
| @PropertySource | 相当于 `<context:property-placeholder>`，引入外部属性配置文件 |
| @Import         | 相当于 `<import resource="">` 引入外部配置类                 |
| @Value          | 对Bean对象变量赋值，可以直接赋值，也可以使用`@${}` 读取配置文件中的信息 |
| @Bean           | 相当于 `<bean>`标签，将方法的返回对象放入IOC 容器中          |

## 1.2、Spring IOC 高级特性

### 1.2.1、lazy-init 延迟加载

`ApplicationConetext` 容器的默认行为是在启动服务器时将所有 `singleton` `bean` 提前进行实例化。提前实例化意味着作为初始化过程的一部分，`ApplicationContext` 实例会创建并配置所有 `singleton` `bean`。

例如：

```xml
<bean id="person" class="com.github.bean.Person"/>
<!-- 该 Bean 默认的设置为 -->
<bean id="person" class="com.github.bean.Person" lazy-init="false"/>
```

设置 `lazy-init` 为 `true` 的 bean 将不会在 `ApplicationContext` 启动时提前被实例化，而是第一次向容器通过 `getBean` 索取 bean 时才进行实例化的。

如果一个设置了立即加载的 Person bean，引用了一个延迟加载的 Address bean，那么 person 在容器启动时候被实例化，而 address 由于被 person 引用，所以也被实例化，这种情况也符合延迟加载的 bean 在第一次被调用时才被实例化的规则。

也在在容器层次中通过在元素上使用 `default-lazy-init` 属性来控制延迟初始化。如下面配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans.xsd"
    default-lazy-init="true">
</beans>
```

如果一个 bean 的 `scope` 属性为 `scope=‘prototype` 时，即便设置了 `lazy-init='false'` ，容器启动时也不会实例化 bean，而是调用 `getBean` 方法实例化的。

- 应用场景
  - 开启延迟加载一定程度上能够提高容器的启动和运行的性能
  - 对于不常使用的 bean 设置延迟加载，这样偶尔使用的时候再加载，不必要从一开始该 bean 就占用资源

### 1.2.2、FactoryBean 和 BeanFactory

`BeanFactory` 接口是容器的顶级接口，定义了容器的一些基础行为，负责生产和管理 Bean 的一个工厂，具体使用它下面的子接口类型，例如 `ApplicationContext`。

在 Spring 中 Bean 有两种，一种是普通 Bean，一种是 `FactoryBean` （工厂Bean），`FactoryBean` 可以生成某一个类型的 Bean实例（返回给我们），也就是我们可以借助它自定义 Bean 的创建过程。

在 Spring 中 [ Bean创建的三种方式](#1.1.2.2、实例化Bean的三种方式 ) 中的 **静态工厂方法**和**实例化工厂方法**，他们与`FactoryBean` 作用类似，`FactoryBean` 使用较多，尤其在 Spring 框架一些组件中会使用，还有其他框架和 Spring 框架整合时使用。

`FactoryBean` 源码：

```java
public interface FactoryBean<T> {
	// 返回 FactoryBean 创建的 Bean实例，如果 isSingleton() 返回的是true,
    // 则该对象将会存放在Spring容器的单例对象缓存池中
	T getObject() throws Exception;
	// 返回 FactoryBean 创建的 Bean 类型
	Class<?> getObjectType();
	// 返回作用域是否是单例
	boolean isSingleton();
}
```

#### 1.2.2.1、FactoryBean 使用

- Student 类

```java
public class Student{
    private String name;
    private String grade;
    // 省略 get/set/toString 方法
}
```

- StudentFactoryBean类

```java
public class StudentFactoryBean implements FactoryBean<Student>{
   private String studentInfo;
   public Student getObject() throws Exception{
       Student stu =  new Student();
       String[]  info = studentInfo.split(",");
       stu.setName(info[0]);
       stu.setGrade(info[1]);
       return stu;
    }  
   public Class<Student> getObjectType(){
       return Student.class;
   }
   public boolean isSingleton(){
       return true;
   }
   // 省略 studentInfo 的 set/get 方法
}
```

- xml 配置

```xml
<bean id="student" class="com.github.factory.bean.StudentFactoryBean">
	<property name="studentInfo" value=" 张三,三年二班"/>
</bean>
```

- 测试程序

```java
// 获取student
Object obj =  applicationContext.getBean("student");
System.out.println("stu:"+obj)
// 获取student工厂,要获取工厂本身需要在id前面加上“&”符号
Object objFactory =  applicationContext.getBean("&student");
System.out.println("objFactory:"+objFactory)
```

- 测试结果

```shell
stu: Student{name="张三",grade="三年二班"}
objFactory:com.github.factory.bean.StudentFactoryBean@26f3fd01
```

### 1.2.3、后置处理器

Spring 框架提供了两种后置处理 Bean 分别是 `BeanPostProcessor` 和 `BeanFactoryPostProcessor`，两者在使用上是有所区别的。

#### 1.2.3.1、BeanPostProcessor

bean 级别的处理，针对某个具体的bean进行处理

```java
public interface BeanPostProcessor {
	Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
	Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```

`BeanPostProcessor` 接口提供了两个方法，分别是初始化前和初始化后执行方法。具体这个初始化方法指的是在定义 Bean 时 ，通过 `init-method` 属性指定的方法 ，例如：

```xml
<bean id="person" class="com.github.bean.Person" init-method="init"/>
```

`BeanPostProcessor` 接口的这两个方法分别在初始化方法前后执行（示例中的init方法被之前前与执行后执行），需要注意的是**当我们定义一个实现了BeanPostProcessor接口的类，默认是会对整个Spring容器中所有的bean进行处理**。

既然是默认全部处理，那么我们怎么确认我们需要处理的某个具体的bean呢？

可以看到方法中有两个参数。类型分别为Object和String，第一个参数是每个 bean 的实例，第二个参数是每个bean 的 name 或者 id 属性的值。所以我们可以第二个参数，来确认我们将要处理的具体的bean。

> 注意：处理是发生在Spring容器的实例化和依赖注入之后。

#### 1.2.3.2、BeanFactoryPostProcessor

BeanFactory 级别的处理，是针对整个Bean的工厂进行处理，典型应用 `PropertyPlaceholderConfigurer`

```java
public interface BeanFactoryPostProcessor {
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```

`BeanFactoryPostProcessor`接口只提供了一个方法,方法参数为 `ConfigurableListableBeanFactory`，其接口定义如下

```java
public interface ConfigurableListableBeanFactory
		extends ListableBeanFactory, AutowireCapableBeanFactory, ConfigurableBeanFactory {
    
	void ignoreDependencyType(Class<?> type);

	void ignoreDependencyInterface(Class<?> ifc);

	void registerResolvableDependency(Class<?> dependencyType, Object autowiredValue);

	boolean isAutowireCandidate(String beanName, DependencyDescriptor descriptor)
			throws NoSuchBeanDefinitionException;

	BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

	Iterator<String> getBeanNamesIterator();

	void clearMetadataCache();

	void freezeConfiguration();

	boolean isConfigurationFrozen();

	void preInstantiateSingletons() throws BeansException;
}
```

其中有个方法名为 `getBeanDefinition` 的方法，我们可以根据此方法，找到我们定义bean的 `BeanDefinition` 对象。然后我们可以对定义的属性进行修改，以下是`BeanDefinition`中的方法

![BeanDefinition定义](04-Spring%20IOC%20%E5%BA%94%E7%94%A8/image-20200717172925002.png)

该接口的定义的属性和方法名字类似我们 bean 标签的属性，setBeanClassName 对应bean标签中的class属性，所以当我们拿到BeanDefinition对象时，我们可以手动修改bean标签中所定义的属性值。

`BeanDefinition` 即我们在xml中定义了bean标签时，Spring会把这些bean标签解析成一个javabean，这个BeanDefinition 就是 bean 标签对应的javabean。它是Spring 内部处理 bean 的数据结构。

Spring容器初始化 bean大致过程   定义bean标签 > 将bean标签解析成BeanDefinition > 调用构造方法实例化(IOC) > 属性值得依赖注入(DI)

所以BeanFactoryPostProcess方法的执行是发生在第二步之后，第三步之前。

> 注意：当我们调用BeanFactoryPostProcess方法时，这时候bean还没有实例化，此时bean刚被解析成BeanDefinition对象。

#### 1.2.3.3、总结

以上两种都为Spring提供的后处理bean的接口，只是两者执行的时机不一样。前者为实例化之后，后者是实例化之前。功能上，后者对bean的处理功能更加强大