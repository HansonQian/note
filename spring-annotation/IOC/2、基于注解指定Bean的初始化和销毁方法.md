# 1、使用注解来处理Bean的生命周期方法

> IOC容器中Bean的生命周期主要包括：Bean的创建--->Bean的初始化--->Bean的销毁 ，这一过程。

## 1.1、Bean的创建

​	调用构造函数创建对象

- 单例对象：IOC容器启动的时候创建单例对象
- 多实例对象：在每次获取的时候创建对象

## 1.2、Bean的初始化

​	对象创建完成并为其属性赋值完成之后，调用初始化方法

## 1.3、Bean的销毁

 - 单实例对象：IOC容器关闭的时候执行Bean的销毁方法
 - 多实例对象：IOC容器不会调用销毁方法

# 2、指定Bean对象初始化方法和销毁方法的四种方式

## 2.1、工程结构

```powershell
com.hanson
	-- bean
	-- config
	- InitAndDestroyApplication.java # 测试程序
```

## 2.2、方式一：使用@Bean注解属性

### 2.2.1、创建Apple.java

```java
public class Apple{
    private String color;
    public Apple() {
        System.out.println("****Apple**** constructor is called ...");
    }
    public Apple(String color) {
        this.color = color;
    }
    public void init() {
        System.out.println("****Apple**** init is called ...");
    }
    public void destroy() {
        System.out.println("****Apple**** destroy is called ...");
    }
    public void setColor(String color) {
        System.out.println("****Apple**** attribute assignment . . . .");
        this.color = color;
    }
}
```

### 2.2.2、创建InitAndDestroyConfiguration.java

```java
@Configuration
public class InitAndDestroyConfiguration {
    @Bean(initMethod = "init", destroyMethod = "destroy")
    public Apple apple() {
        Apple apple = new Apple();
        apple.setColor("red");
        return apple;
    }
}
```

### 2.2.3、测试程序

```java
public class InitAndDestroyApplication {
    public static void main(String[] args) {
    AnnotationConfigApplicationContext context = new 		AnnotationConfigApplicationContext(InitAndDestroyConfiguration.class);
 //遍历全部的Bean名称       Arrays.asList(context.getBeanDefinitionNames()).forEach(System.out::println);
    context.close();
    }
}
```

### 2.2.4、运行结果

```powershell
****Apple**** constructor is called ...
****Apple**** attribute assignment . . . .
****Apple**** init is called ...
****Apple**** destroy is called ...
```

## 2.3、方式二：实现InitializingBean和DisposableBean接口

- 实现接口`InitializingBean`覆写`afterPropertiesSet`方法,该方法会在当前Bean对象创建成功并为属性赋值完成之后调用
- 实现接口`DisposableBean`覆写destroy方法,该方法会在容器关闭的时候调用(**单例对象有效**)

### 2.3.1、创建Blueberry.java

> 实现Spring提供的接口 实现InitializingBean和DisposableBean接口

```java
public class Blueberry implements InitializingBean, DisposableBean {
    private String color;
    public Blueberry() {
       System.out.println("****Blueberry**** constructor is called ...");
    }
    public Blueberry(String color) {
        this.color = color;
    }
    public void setColor(String color) {
        System.out.println("****Blueberry**** attribute assignment . . . .");
        this.color = color;
    }
    @Override
    public void destroy() throws Exception {
        System.out.println("****Blueberry**** destroy is called ...");
    }
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("****Blueberry**** afterPropertiesSet is called ...");
    }
}
```

### 2.3.2、创建InitAndDestroyConfiguration.java

```java
@Configuration
public class InitAndDestroyConfiguration{
    @Bean
    public Blueberry blueberry() {
        Blueberry blueberry = new Blueberry();
        blueberry.setColor("Blue");
        return blueberry;
    }
}
```

### 2.3.3、测试程序

略

### 2.3.4、运行结果

```powershell
**** Blueberry constructor is called ...
**** Blueberry attribute assignment...
**** Blueberry afterPropertiesSet is called ...
**** Blueberry destroy is called ...
```

## 2.4、方式三：使用JSR250规范提供的两个注解

- 注解`@PostConstructor`标注在初始化方法上,该方法会在当前Bean对象创建成功并为属性赋值完成之后调用
- 注解`@PreDestory`标注在销毁方法上,该方法会在容器关闭的时候调用(**单例对象有效**)

### 2.4.1、创建Pear.java

```java
public class Pear {
    private String color;
    public Pear() {
        System.out.println("####JSR250规范 Pear constructor is called ...");
    }
    public Pear(String color) {
        this.color = color;
    }
    public void setColor(String color) {
        System.out.println("####JSR250规范 Pear attribute assignment . . .");
        this.color = color;
    }
    @PostConstruct
    public void init() {
        System.out.println("####JSR250规范 Pear init is called . . .");
    }
    @PreDestroy
    public void destroy() {
        System.out.println("####JSR250规范 Pear destroy is called . . .");
    }
}
```

### 2.4.2、创建InitAndDestroyConfiguration.java

```java
@Configuration
public class InitAndDestroyConfiguration{
    @Bean
    public Pear pear() {
        Pear pear = new Pear();
        pear.setColor("Yellow");
        return pear;
    }
}
```

### 2.4.3、测试程序

略

### 2.4.4、运行结果

```powershell
####JSR250规范 Pear constructor is called ...
####JSR250规范 Pear attribute assignment . . .
####JSR250规范 Pear init is called . . .
####JSR250规范 Pear destroy is called . . .
```

## 2.5、方式四：实现BeanPostProcessor接口

> IOC容器会首先遍历容器中全部的后处理Bean接口实现类;然后挨个执行postProcessBeforeInitialization方法；在执行过程中一旦遇到返回null，则会调出循环，即而不会继续执行后面的postProcessBeforeInitialization方法

- 覆写方法`postProcessBeforeInitialization`,该方法会在调用初始化方法之前调用
- 覆写方法`postProcessAfterInitialization`,该方法会在调用完初始化方法之后调用 

### 2.5.1、创建Orange.java

```java
public class implements BeanPostProcessor {
    private String color;
    public Orange() {
        System.out.println("BeanPostProcessor Orange constructor is called");
    }
    public Orange(String color) {
        this.color = color;
    }
    public void setColor(String color) {
       System.out.println("BeanPostProcessor Orange attribute assignment");
        this.color = color;
    }
    public void init() {
        System.out.println("BeanPostProcessor Orange init is called");
    }
    public void destroy() {
        System.out.println("BeanPostProcessor Orange destroy is called ");
    }
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("****在调用初始化方法之前调用 ...");
        return bean;
    }
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("****在调用初始化方法之后调用 ...");
        return bean;
    }   
}
```

### 2.5.4、创建InitAndDestroyConfiguration.java

```java
@Configuration
public class InitAndDestroyConfiguration{
    @Bean(initMethod = "init", destroyMethod = "destroy")
    public Orange orange() {
        Orange orange = new Orange();
        orange.setColor("Orange");
        return orange;
    }
}
```

### 2.5.3、测试程序

略

### 2.5.4、运行结果

此处未能实现如下效果，可能与`BeanPostProcessor`使用方式有关

```powershell
****Implement BeanPostProcessor **** Orange constructor is called ...
****在调用初始化方法之前调用 ...
****Implement BeanPostProcessor **** Orange attribute assignment ...
****在调用初始化方法之后调用 ...
****Implement BeanPostProcessor **** Orange init is called ...
****Implement BeanPostProcessor **** Orange destroy is called ...
```

# 3、BeanPostProcessor源码分析

- IOC容器创建Bean之后，调用`populateBean(beanName, mbd, instanceWrapper)`方法;给bean进行属性赋值
- 调用Bean的初始化方法`initializeBean`
  - 调用`applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);`
  - 调用`invokeInitMethods(beanName, wrappedBean, mbd);`执行自定义初始化
  - `applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);`

## 3.1、后处理Bean在Spring中的应用

- 给Bean赋值
- 注入其他组件
- 实现依赖注入@Autowired
- Bean生命周期注解功能
- @Async异步执行注解
- 若干XXXBeanPostProcessor