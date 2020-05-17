# 一、Spring Boot 与Web开发

## 1、Spring MVC框架设计

### 1.1、Spring MVC架构

![image-20200312160016197](04%E3%80%81Spring%20Boot%E4%B8%8EWeb%E5%BC%80%E5%8F%91/image-20200312160016197.png)

### 1.2、Spring MVC架构流程图

1. 用户向服务器发送请求，请求被Spring前端控制器(DispatcherServlet)捕获
2. 前端控制器会对请求的url进行解析，得到请求资源的标识符(uri)，然后根据uri，调用处理器映射器(HandlerMapping)获得该处理器(Handler)配置的所有相关的对象（包括Handler对象以及Handler对象对应的拦截器），最后以HandlerExecutionChain对象的形式返回。
3. 前端控制器根据获得的Handler，会选择一个合适的处理器适配器(HandlerAdapter)。在这之前如果成功获得了处理器适配器，那么会先执行拦截器里面的preHandler(...)方法。
4. 提取request域对象中的模型数据，填充至处理器中具体目标方法的形参中。在填充的过程中会根据mvc-dispatcher-servlet.xml中配置进行一些额外的处理：
   - HttpMessageConveter：将请求信息(例如Json格式或者是xml格式的请求数据)转换成一个对象，当处理完也可以将处理结果，将对象转换为指定的响应信息
     - 数据转换：对请求信息进行数据转换，例如将String转为Integer、Double等
     - 数据格式化：对请求信息进行数据格式化，例如将String类型转为Date类型
     - 数据验证：验证数据的有效性(参数校验)，长度，是否非空等验证结果存储到BindingResult或ObjectError中
5. 处理器执行完之后会向前端控制器返回一个ModelAndView对象
6. 根据处理器返回的ModelAndView对象，前端控制器会对其进行解析为Model和View，然后根据View选择一个合适的并且已经在Spring容器中注册的视图解析器(ViewReslover)
7. 视图解析器会结合Model和View（将model数据填充到request域对象中）来进行渲染视图
8. 最后将渲染的结果在客户端展示

## 2、Web开发示例

​		Spring Boot对Web开发的支持很全面，包括开发、测试和部署阶段都做了支持。`spring-boot-starter-web`是对Web开发提供支持的组件，主要包括RESTful，参数校验，使用Tomcat作为内嵌容器等等。

### 2.1、JSON的支持

​		JSON 是一种轻量级数据交互格式，易于阅读和编写，同时也易于机器解析和生成。

### 2.2、创建一个Web项目

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
</dependencies>
```

添加spring boot对web开发支持的组件依赖

### 2.3、创建Model类

```java
@Data
@Accessors(chain = true)
public class User {
    private Long id;
    private String userName;
    private String note;
}
```

### 2.4、创建Controller

```java
@RestController
public class UserController {
    @GetMapping("/user/{id}")
    public User getUser(@PathVariable("id") long id) {
        return new User().setId(id).setUserName("Demo").setNote("Spring Boot");
    }
}
```

- `@RestController`相当于`@Controller`与`@ResponseBody`的组合，该注解如果作用在类上面，说明该类的全部方法都会以JSON的数据格式返沪
- `@GetMapping`相当于`@RequestMapping(value = "/user/{id}",method = RequestMethod.GET)`，除此之外还有
  - `PostMapping`
  - `DeleteMapping`
  - `PutMapping`

### 2.5、创建启动程序

```java
@SpringBootApplication
public class ExampleWebApplication{
    public static void main(String[] args) {
        SpringApplication.run(ExampleWebApplication.class, args);
    }
}
```