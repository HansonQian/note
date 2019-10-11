# Java Web 知识点

- HTML

- CSS&JS

  - JQuery

- MySQL

  - 熟练使用DDL以及DML语句
  - 参考：<https://www.cnblogs.com/yangyangfubin/p/8179172.html>

- JDBC

  - 熟练掌握JDBC连接数据库步骤
  - 掌握ResultSet、PreparedStatement、Statement对象
    - PreparedStatement对比Statement的优点
  - 事务的概念
    - ACID四大特性
    - 脏读、幻读、不可重复读
    - 数据的默认隔离级别
      - MySQL
      - Oracel
  - 熟练掌握使用JDBC控制事务
  - 使用数据库连接池
    - C3P0
    - DBCP
    - Druid

- XML&Tomcat

  - XML

    - 概念
    - 语法
    - 解析
      - JDK 提供的API
      - 第三方API

  - Tomcat

    - 软件架构

      - C/S:客户端/服务器端
        - QQ、迅雷
      - B/S:浏览器/服务器端
        - 淘宝、京东

    - 资源分类

      - 静态资源:所有用户访问后，得到的结果都是一样的，称为静态资源.静态资源可以直接被浏览器解析
        - html,css,JavaScript等
      - 动态资源:每个用户访问相同资源后，得到的结果可能不一样。称为动态资源。动态资源被访问后，需要先转换为静态资源，在返回给浏览器
        - servlet/jsp,php,asp等

    - 网络通信三要素

      - IP：电子设备(计算机)在网络中的唯一标识
      - PORT：应用程序在计算机中的唯一标识。取值范围（ 0~65536）
      - PROTOCOL： 规定了数据传输的规则
        - 基础协议：
          - TCP：安全协议，三次握手。 速度稍慢
          - UDP：不安全协议。 速度快

    - Web服务器软件

      - 服务器：安装了服务器软件的计算机

      - 服务器软件：接收用户的请求，处理请求，做出响应

      - web服务器软件：接收用户的请求，处理请求，做出响应。

        - 在web服务器软件中，可以部署web项目，让用户通过浏览器来访问这些项目
        - web容器

      - 常见的java相关的web服务器软件

        - webLogic：oracle公司，大型的JavaEE服务器，支持所有的JavaEE规范，收费
        - webSphere：IBM公司，大型的JavaEE服务器，支持所有的JavaEE规范，收费
        - JBOSS：JBOSS公司的，大型的JavaEE服务器，支持所有的JavaEE规范，收费
        - Tomcat：Apache基金组织，中小型的JavaEE服务器，仅仅支持少量的JavaEE规范servlet/jsp。开源免费

      -  JavaEE：Java语言在企业级开发中使用的技术规范的总和，一共规定了13项大的规范

      - Tomcat：

        - web服务器软件

        - 下载：http://tomcat.apache.org/

        - 安装：解压压缩包即可。

          - 注意：安装目录建议不要有中文和空格

        - 卸载：删除目录

        -  启动：

          - bin/startup.bat ,双击运行该文件即可

        - 访问：http://localhost:8080

        - 关闭：

          - 正常关闭
            - bin/shutdown.bat
            - ctrl+c
          - 强制关闭
            - 关闭小黑窗

        - 配置

          - 部署项目的方式：

            - 直接将项目放到webapps目录下即可

            -  简化部署：

              - 将项目打成一个war包，再将war包放置到webapps目录下
                -  war包会自动解压缩

            - 配置conf/server.xml文件

              - 在<host>标签体中配置
                - `<Context docBase="D:\hello" path="/hehe" />`
                  - docBase:项目存放的路径
                  - path：虚拟目录

            - 静态项目和动态项目

              - 目录结构

                - java动态项目的目录结构

                  ```shell
                  --项目根目录
                  ----WEB-INF目录:
                  ------web.xml：web项目的核心配置文件
                  ------classes目录：放置字节码文件的目录
                  ------lib目录：放置依赖的jar包
                  ```

            - 将Tomcat集成到IDEA中，并且创建JavaEE的项目，部署项目

- Servlet

  - 概念：运行在服务器端的小程序

    - Servlet就是一个接口，定义了Java类被浏览器访问到(tomcat识别)的规则
    - 自定义一个类，实现Servlet接口，复写方法

  - 入门程序

    - 自定义一个类实现Servlet接口并复写其中的抽象方法

      ```java
      public HelloWorldServlet implements Servlet｛｝
      ```

    - 配置Servlet

      ```xml
      <servlet>
      	<servlet-name>HelloWorldServlet</servlet-name>
      	<servlet-class>com.ls.web.HelloWorldServlet</servlet-class>
      </servlet>
       <servlet-mapping>
      	<servlet-name>HelloWorldServlet</servlet-name>
      	<url-pattern>/hello</url-pattern>
      </servlet-mapping>
      ```

      在web.xml中配置如上信息

    - 执行原理（重要）

      ```shell
      1. 当服务器接受到客户端浏览器的请求后，会解析请求URL路径，获取访问的Servlet的资源路径
      2. 查找web.xml文件，是否有对应的<url-pattern>标签体内容。
      3. 如果有，则在找到对应的<servlet-class>全类名
      4. tomcat会将字节码文件加载进内存，并且创建其对象
      5. 调用其方法
      ```

    - Servlet中的生命周期方法(重要)

      - 1、被创建：执行init方法，只执行一次

        - Servlet什么时候被创建？

          - 默认情况下，第一次被访问时，Servlet被创建

          - 可以配置执行Servlet的创建时机

            ```xml
            <servlet>
            	<servlet-name>HelloWorldServlet</servlet-name>
            	<servlet-class>...HelloWorldServlet</servlet-class>
                <load-on-startup>1</load-on-startup>
            </servlet>
            ```

            `load-on-startup`配置值为负数：第一次访问时被创建

            `load-on-startup`配置值为正数或者0：在服务器启动时创建

        - Servlet的init方法，只执行一次，说明一个Servlet在内存中只存在一个对象，Servlet是单例的

          - 多个用户同时访问时，可能存在线程安全问题
          - 解决：尽量不要在Servlet中定义成员变量。即使定义了成员变量，也不要对修改值

      - 2、提供服务：执行service方法，执行多次

        - 每次访问Servlet时，Service方法都会被调用一次

      - 3、被销毁：执行destroy方法，只执行一次

        - Servlet被销毁时执行。服务器关闭时，Servlet被销毁
        - 只有服务器正常关闭时，才会执行destroy方法。
        - destroy方法在Servlet被销毁之前执行，一般用于释放资源

    - Servlet 3.0版本

      - 提供注解，可以省略web.xml配置

      - 入门步骤

        - 创建J2EE项目,选择Servlet的版本3.0以上,可以不用创建web.xml

        - 定义一个类，实现Servlet接口

        - 复写方法

        - 在类上使用`@WebServlet`注解，进行配置

          ```java
          @Target({ElementType.TYPE})
          @Retention(RetentionPolicy.RUNTIME)
          @Documented
          public @interface WebServlet {
              String name() default "";//相当于<Servlet-name>
              String[] value() default {};//代表urlPatterns()属性配置
              String[] urlPatterns() default {};//相当于<url-pattern>
              int loadOnStartup() default -1;//相当于<load-on-startup>
          	...
          }
          ```

        - 启动访问测试

    - Servlet体系结构

      ```java
      Servlet #接口
      ----GenericServlet #抽象类
      --------HttpServlet  #抽象类
      ```

      GenericServlet：将Servlet接口中其他的方法做了默认空实现，只将service()方法作为抽象，可以继承GenericServlet，实现service()方法即可

      HttpServlet：对http协议的一种封装，简化操作

      ​	定义类继承HttpServlet

      ​	复写doGet/doPost方法

    - Servlet相关配置

      urlpartten:Servlet访问路径

      一个Servlet可以定义多个访问路径 ： @WebServlet({"/d4","/dd4","/ddd4"})

      路径定义规则：
      1. /xxx：路径匹配
      2. /xxx/xxx:多层路径，目录结构
      3. *.do：扩展名匹配

- Request&Response

- Cookie&Session

- JSP

- Java Web设计模式（主要是MVC设计模式）

- 事务&数据库连接池

- DBUtils工具类使用

- 分页

- Listener&Filter

- 文件上传&下载

- Java基础扩展
  - 反射
  - 枚举
  - 泛型
  - 注解