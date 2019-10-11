# 1、Spring Boot集成Mybatis

## 1.1、集成通用Mapper及PageHelper

### 1.1.1、添加信息

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.4.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.hanson</groupId>
    <artifactId>user-service</artifactId>
    <packaging>pom</packaging>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <mapper.starter.version>2.1.5</mapper.starter.version>
        <mysql.version>5.1.46</mysql.version>
        <pageHelper.starter.version>1.2.5</pageHelper.starter.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.4</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
        <!--通用mapper 依赖-->
        <dependency>
            <groupId>tk.mybatis</groupId>
            <artifactId>mapper-spring-boot-starter</artifactId>
            <version>${mapper.starter.version}</version>
        </dependency>
        <!--mybatis分页助手 依赖-->
        <dependency>
            <groupId>com.github.pagehelper</groupId>
            <artifactId>pagehelper-spring-boot-starter</artifactId>
            <version>${pageHelper.starter.version}</version>
        </dependency>        
        <!--mysql连接驱动 依赖-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>${mysql.version}</version>
        </dependency>
    </dependencies>
    <build>
        <finalName>spring-cloud-parent</finalName>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>
            </resource>
        </resources>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### 1.1.2、数据库脚本信息

```sql
DROP TABLE IF EXISTS `t_user`;
CREATE TABLE `t_user`  (
  `ID` int(12) NOT NULL AUTO_INCREMENT,
  `USER_NAME` varchar(60) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `SEX` int(3) NOT NULL DEFAULT 1,
  `NOTE` varchar(256) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  PRIMARY KEY (`ID`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 2 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Compact;
```

### 1.1.3、编写代码

#### 1.1.3.1、编写SexEnum枚举类

```java
@Getter
@AllArgsConstructor
public enum SexEnum {
    MALE(1, "男"),
    FEMALE(2, "女");
    private int id;
    private String name;
    public static SexEnum getEnumById(int id) {
        for (SexEnum sexEnum : SexEnum.values()) {
            if (sexEnum.getId() == id) {
                return sexEnum;
            }
        }
        return null;
    }
    public void setName(String name) {
        this.name = name;
    }
    public void setId(int id) {
        this.id = id;
    }
}
```

#### 1.1.3.2、编写User类

```java
@Setter
@Getter
@ToString
@Table(name = "t_user")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(name = "user_name")
    private String userName;
    private String note;
    @Column(name = "sex")
    @ColumnType(jdbcType = JdbcType.INTEGER, typeHandler = SexEnumTypeHandler.class)
    private SexEnum sex;
}
```

#### 1.1.3.3、自定义类型处理器

```java
public class SexEnumTypeHandler extends BaseTypeHandler<SexEnum> {
    //设置非空性别参数
    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, SexEnum parameter, JdbcType jdbcType) throws SQLException {
        ps.setInt(i, parameter.getId());
    }
    //通过列名读取
    @Override
    public SexEnum getNullableResult(ResultSet rs, String columnName) throws SQLException {
        int sex = rs.getInt(columnName);
        return getEnumById(sex);
    }
    //通过下标读取
    @Override
    public SexEnum getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        int sex = rs.getInt(columnIndex);
        return getEnumById(sex);
    }
    //通过存储过程读取
    @Override
    public SexEnum getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        int sex = cs.getInt(columnIndex);
        return getEnumById(sex);
    }
    private SexEnum getEnumById(int sex) {
        if (sex != 1 && sex != 2) {
            return null;
        }
        return SexEnum.getEnumById(sex);
    }
}
```

#### 1.1.3.4、编写映射器

```java
public interface UserMapper extends Mapper<User> {
}
```

#### 1.1.3.5、编写配置类

```java
@Configuration
public class BeanConfiguration {
    @Bean
    public ConfigurationCustomizer configurationCustomizer() {
        return configuration -> {
            //开启驼峰自动转换
            configuration.setMapUnderscoreToCamelCase(true);
            //别名处理器
            TypeAliasRegistry typeAliasRegistry = 
                configuration.getTypeAliasRegistry();
            typeAliasRegistry.registerAliases("spring.boot.mybatis.pojo");
            //类型处理器
            TypeHandlerRegistry typeHandlerRegistry = 
                configuration.getTypeHandlerRegistry();
            typeHandlerRegistry.register(SexEnum.class, 
                                         new SexEnumTypeHandler());
            //添加分页插件
            PageInterceptor pageInterceptor = new PageInterceptor();
            Properties pageInterceptorProperties = new Properties();
            pageInterceptorProperties.setProperty("helperDialect", "mysql");
            //设置为true时，使用RowBounds分页会进行count查询
            pageInterceptorProperties.setProperty("rowBoundsWithCount", "true");
            pageInterceptor.setProperties(pageInterceptorProperties);
            configuration.addInterceptor(pageInterceptor);
            //添加通用mapper映射器
            configuration.addMapper(tk.mybatis.mapper.common.Mapper.class);
        };
    }
}
```



#### 1.1.3.6、编写启动程序

```java
@SpringBootApplication
@EnableDiscoveryClient
@MapperScan("com.hanson.user.service.mapper")
public class UserServiceApplication_8080 {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication_8080.class, args);
    }
}
```

注意：`@MapperScan`是`tk.mybatis.spring.annotation`包下面的