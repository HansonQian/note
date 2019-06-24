### 1、服务注册与发现

#### 1.1、Eureka使用

##### 1.1.1、创建maven工程

创建一个maven工程名字为:`eureka-server-7001`，`pom.xml`文件内容如下:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>eureka-modules</artifactId>
        <groupId>com.hanson</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>
    <artifactId>eureka-server-7001</artifactId>
    <name>eureka-server-7001 :: 服务注册中心</name>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-parent</artifactId>
                <version>1.3.7.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Brixton.SR5</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka-server</artifactId>
        </dependency>
    </dependencies>
</project>
```

##### 1.1.2、编写程序

`EurekaServerApplication_7001.java`内容如下:

```java
package com.hanson.eureka.server;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;
@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApplication_7001 {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication_7001.class);
    }
}
```

注:`使用@EnableEurekaServer`表明当前应用是Eureka注册中心组件的服务端

`application.yml`配置内容如下:

```yaml
server:
  port: 7001
spring:
  application:
    name: eureka-server
eureka:
  instance:
    hostname: localhost 
    prefer-ip-address: true
  client:
    fetch-registry: false
    register-with-eureka: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

说明:

- `server.port`：指定运行的端口号(SpringBoot嵌入式Web容器)
- `spring.application.name`：指定运行应用的名称
- `eureka.instance.hostname`：指定eureka实例的主机名
- `eureka.instance.prefer-ip-address`：将IP注册到Eureka Server上，如果不配置就是机器的主机名。

- `eureka.client.fetch-registry`：表示是否从Eureka server获取注册信息，如果是单一节点，不需要同步其他Eureka server节点，则可以设置为false，如果为集群，那么应该设置为true，默认为true，可不设置。

- `eureka.client.register-with-eureka`：表示是否将自己注册到Eureka server，如果要构建集群环境，那么需要将自己注册到及群众，则应该开启。默认为true，可不显式设置。如果构建的是单节点，那么需要配置成false
- `eureka.client.service-url.defaultZone`：设置与Eureka Server交互的地址,查询服务和注册服务都需要依赖这个地址.默认是`http://localhost:8761/eureka/`多个地址可以使用半角逗号分割

##### 1.1.3、测试

访问地址:http://loclahost:7001/，能够正常访问Eureka主页说明程序已经启动成功

##### 1.1.4、Eureka高可用

查看工程`eureka-server-7002`和`eureka-server-7003`两个工程

与`eureka-server-7001`不同的地方在配置文件