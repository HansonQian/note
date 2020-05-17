# 一、Spring Boot配置文件

## 1、配置文件类型

### 1.1、配置文件类型和作用

​		Spring Boot是基于约定的，所以很多配置都是有默认值，但是如果想要使用自己的配置替换默认配置的话，就可以使用application.properties或者application.yml（application.yaml）进行配置。

​		Spring Boot默认会从Resource目录下加载applicayion.properties或application.yml文件。其中application.properties文件是键值对类型的文件，这里不再叙述。

## 2、YAML配置文件

### 2.1、YML配置文件简介

​		YML文件格式是YAML(YAML Aint Markup Language)编写的文件格式，YAML是一种直观的能够被电脑识别的数据序列化格式，并且容易被人类阅读，容易和脚本语言交互的，可以被支持YAML库的不同的编程语言程序导入，比如： C/C++, Ruby, Python, Java, Perl, C#, PHP等。YML文件是以数据为核心的，比传统的xml方式更加简洁。

​		YML文件的扩展名可以使用.yml或者.yaml。

### 2.2、YML语法

#### 2.2.1、基本语法

- 使用缩进表示层级关系
- 缩进时不允许使用tab键，只允许使用空格
- 缩进的空格数目不重要，只要相同层级的元素左侧对齐即可
- 大小写敏感
- 在线语法校验
  - https://nodeca.github.io/js-yaml/

#### 2.2.2、YML支持三种数据结构

- 对象：键值对的集合
- 数组：一组按次序排列的值
- 字面量：单个的、不可再分的值
- 复合结构：整合上述3种类型

#### 2.2.3、常用写法

- 对象

  - 对象的一组键值对，使用冒号分隔

  - 冒号后面跟空格来分开键值

  - {k: v}是行内写法，注意空格

  - 示例

    ```yaml
    person:
    	name: 张三
    	age: 24
    	addr: 无锡
    ```

    或者

    ```yaml
    person: {name: 张三,age: 24,addr: 无锡}
    ```

- 数组

  - 一组连词先(-)开头的行，构成一个数组，[]为行内写法

  - 数组和对象可以组合使用

  - 示例

    ```yaml
    city:
        - 无锡
        - 苏州
        - 南京
    ```

    或者

    ```yaml
    city: [无锡,苏州,南京]
    ```

- 字面量

  - 数字、字符串、布尔、日期
  - 字符串
    - 默认不使用引号
    - 可以使用单引号或者双引号，单引号会转义特殊字符
    - 字符串可以写成多行，从第二行开始，必须有一个单空格缩进。换行符会被转为空格。

- 复合结构

  - 示例

    - Java程序

      ```java
      @Component
      @ConfigurationProperties(prefix = "person")
      public class Person {
          private String name;
          private String username;
          private Integer age;
          private Map<String, Object> pet;
          private List<String> animal;
          private List<String> interests;
          private List<Object> friends;
          private List<Map<String, Object>> childs;
         //省略Getter/Setter方法
      ```

    - YML写法

      ```yaml
      person:
        name: 'ZhangSan \n'
        username: 张三
        age: 18
        pet:
          name: 小狗
          gender: male
        animal:
          - dog
          - cat
          - fish
        interests: [足球、篮球]
        friends:
           - ZhangSan is my best firend
           - "LiSi\n"
        childs:
          - name: xiaozhang
            age: 18
          - name:
             pets:
               - a
               - b
          - {name: LiSi, age: 18}
      ```

> Spring Boot 使用`snakeyaml`解析YAML文件

## 3、配置文件值注入

### 3.1、配置文件加载

​		使用注解`@PropertySource`与`@ImportSource`来加载配置文件

- 使用`@PropertySource`加载指定的配置文件

- 配和`@ConfigurationProperties`注解，告诉Spring Boot将本类中的所有属性和配置文件中相关的配置进行绑定

  ```java
  @Target({ElementType.TYPE, ElementType.METHOD})
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  public @interface ConfigurationProperties {
      @AliasFor("prefix")
      String value() default ""; // 配置文件中具体KEY下面进行映射
      @AliasFor("value")
      String prefix() default "";
      boolean ignoreInvalidFields() default false;
      boolean ignoreUnknownFields() default true;
  }
  ```

- 可以使用`@PropertySources`来管理`@PropertySource`

- 使用`@ImportSource`导入Spring配置文件（application.xml）

### 3.2、为Bean属性注值

​	`@Value`和`@ConfigurationProperties`为Bean属性注值对比

|                | `@ConfigurationPrpperties` | `@Value` |
| :------------: | :------------------------: | :------: |
|      功能      |   为Bean的属性批量注入值   | 单个注值 |
|    松散绑定    |            支持            |  不支持  |
|      SpEL      |           不支持           |   支持   |
| JSR303数据校验 |            支持            |  不支持  |
|  复杂类型封装  |            支持            |  不支持  |

- 当某个业务需要获取配置文件中的某一项值使用`@Value`
- 当需要专门为配置文件创建一个JavaBean，使用`@ConfigurationProperties`

##  4、配置文件占位符

### 4.1、随机数

```yaml
${random.value}
${random.int}${random.long}
${random.int(10)}
${random.int[1024,65536]}
```

### 4.2、获取配置文件中的值

```yaml
app.name=MyApplication
app.description=${app.name:Example} is a Spring Boot Application! 
```

- 可以在配置文件中引用前面配置过的属性（优先级：前面配置过的这里都能用）
- ` ${app.name:默认值}`来指定找不到属性时的默认值

## 5、Profile

### 5.1、简介

​		Profile是Spring对不同环境提供不同配置功能的支持，可以通过激活、指定参数等方式快速切换环境。Spring Profiles提供了一种隔离应用程序配置部分并使其仅在特定环境中可用的方法。任何`@Component`或`@Configuration` 可以标记`@Profile`以限制何时加载。

### 5.2、Profile模式类型

- 多profile文件

  - 格式：application-{profile}.yml
  - 例如：application-dev.yml、application-prod.yml

- 多profile文档块模式

  ```yaml
  spring:
    profiles:
      active: default
  
  ---
  spring:
    profiles:
      active: dev
  
  ---
  spring:
    profiles:
      active: test
  
  ---
  spring:
    profiles:
      active: prod
  ```

  使用`---` 来分割多个Profile文档块

### 5.3、Profile激活方式

- 命令行启动
  - 例如：` --spring.profiles.active=dev`
- 配置文件方式
  - 例如：`spring.profiles.active=dev`
- JVM参数方式
  - 例如：`–Dspring.profiles.active=dev`

## 6、配置文件的加载位置

​		Spring Boot 启动会扫描以下位置的`application.properties`或者`application.yml`文件作为Spring Boot的默认配置文件。

- `file:./config/`
- `file:./`
- ` classpath:/config/`
- ` classpath:/`

​      以上是按照优先级从高到低的顺序，所有位置的文件都会被加载，**高优先级配置内容会覆盖低优先级配置内，容**，Spring Boot会从这四个位置全部加载主配置文件，**互补配置**

> 可以通过配置`spring.config.location`来改变默认配置

## 7、配置文件加载顺序

Spring Boot 支持多种外部配置方式，这些方式的优先级参考[地址](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-external-config)

- 命令行参数
- 来自`java:comp/env`的JNDI属性
- Java系统属性`System.getProperties()`
- 操作系统环境变量
- `RandomValuePropertySource`配置的random.*属性值
- jar包外部的`application-{profile}.properties`或`application.yml`(带spring.profile)的配置文件
- jar包内部的`application-{profile}.properties`或`application.yml`(带spring.profile)的配置文件
- jar包外部的`application.properties`或`application.yml`(不带spring.profile)的配置文件
- jar包内部的`application.properties`或`application.yml`(不带spring.profile)的配置文件
- `@Configuration`注解类上的`@PropertySource`
- 通过`SpringApplication.setDefaultProperties`指定的默认属性