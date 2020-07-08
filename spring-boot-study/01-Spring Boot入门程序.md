# 一、Spring Boot入门

##  1、Spring Boot 简介

​		Spring Boot是由Pivotal团队在2013年开始研发、2014年4月发布第一个版本的全新开源的轻量级框架。它基于Spring4.0设计，不仅继承了Spring框架原有的优秀特性，而且还通过简化配置来进一步简化了Spring应用的整个搭建和开发过程。另外Spring Boot通过集成大量的框架使得依赖包的版本冲突，以及引用的不稳定性等问题得到了很好的解决。

​		Spring Boot 可以构建一切。Spring Boot 设计之初就是为了最少的配置，以最快的速度来启动和运行 Spring 项目。Spring Boot 使⽤特定的配置来构建生产就绪型的项⽬。

##  2、背景

​		J2EE笨重的开发、繁多的配置、低下的开发效率、复杂的部署流程、第三方集成难度大

##  3、解决

- Spring 全家桶时代
- Spring Boot提供J2EE一站式解决方案
- Spring Cloud提供分布式整体解决方案

## 4、优点

- 快速创建独立运行的Spring项目以及主流框架集成
- 使用嵌入式的Servlet容器,应用无需打成`war`包
- 提供自动配置的“starter”项目对象模型（POMS）以简化[Maven](https://baike.baidu.com/item/Maven/6094909)配置
- 尽可能自动配置Spring容器
- 提供准备好的特性，如指标、健康检查和外部化配置
- 无需配置XML,无代码生成,开箱即用
- 与云计算天然集成

## 5、入门程序

### 5.1、开发环境约束

- JDK 1.8
- Maven 3.6.3
- IntelliJ IDEA 2019
- Spring Boot 2.2.4.RELEASE

### 5.2、创建Maven工程

​		利用IDEA工具创建一个Maven项目

### 5.3、添加项目依赖

​		在项目的pom.xml中添加GAV

```xml
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.4.RELEASE</version>
    </parent>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
```

### 5.4、创建主程序

```java
package spring.boot.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class ExampleApplication {

    @GetMapping("/hello")
    public String hello() {
        return "Hello Spring Boot!";
    }

    public static void main(String[] args) {
        SpringApplication.run(ExampleApplication.class, args);
    }
}
```

### 5.5、启动运行

​		运行main方法

### 5.6、访问测试

​		通过浏览器访问`http://localhost:8080/hello`即可输出`Hello Spring Boot！`

### 5.7、构建配置

​		在pom文件中添加如下内容

```xml
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
```

​			spring-boot-maven-plugin能够以 Maven 的方式为应⽤用提供 Spring Boot 的支持，即为 Spring Boot 应用提供了执⾏ Maven 操作的可能。spring-boot-maven-plugin 能够将 Spring Boot 应⽤用打包为可执⾏的 jar 或 war 文件，然后以简单的运⾏ Spring Boot 应⽤。