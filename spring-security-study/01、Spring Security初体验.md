# 01、Spring Security 初体验

​		首先需要为Spring Security建立一个上下文配置文件，该文件专门用来作为Spring Security的配置。使用Spring Security需要引入Spring Security的命名空间。

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:security="http://www.springframework.org/schema/security"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans
          http://www.springframework.org/schema/beans/spring-beans-3.1.xsd
          http://www.springframework.org/schema/security
          http://www.springframework.org/schema/security/spring-security-3.1.xsd">
</beans>
```

