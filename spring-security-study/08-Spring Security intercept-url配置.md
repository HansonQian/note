# 1、Spring Security Intercept-url配置

## 1.1、指定拦截的URL

通过pattern指定当前intercept-url定义应当作用于哪些url。

```xml
<security:intercept-url pattern="/**" access="ROLE_USER"/>
```

## 1.2、指定访问权限

可以通过access属性来指定intercept-url对应URL访问所应当具有的权限。access的值是一个字符串，其可以直接是一个权限的定义，也可以是一个表达式。常用的类型有简单的角色名称定义，多个名称之间用逗号分隔，如：

```xml
<security:intercept-url pattern="/secure/**" access="ROLE_USER,ROLE_ADMIN"/>
```

在上述配置中就表示secure路径下的所有URL请求都应当具有`ROLE_USER`或`ROLE_ADMIN`权限。当access的值是以`ROLE_`开头的则将会交由RoleVoter进行处理。

此外，其还可以是一个表达式，上述配置如果使用表达式来表示的话则应该是如下这个样子。

```xml
<security:http use-expressions="true">
    <security:form-login />
    <security:logout />
    <security:intercept-url pattern="/secure/**"access="hasAnyRole('ROLE_USER','ROLE_ADMIN')"/>
</security:http>
```

或者是使用 hasRole()表达式，然后中间以 or 连接，如：

```xml
   <security:intercept-url pattern="/secure/**" access="hasRole('ROLE_USER') or hasRole('ROLE_ADMIN')"/>
```

需要注意的是使用表达式时需要指定 http 元素的 use-expressions=”true”。当 intercept-url 的 access 属性使用表达式时默认将使用 WebExpressionVoter 进行处理。

此外，还可以指定三个比较特殊的属性值，默认情况下将使用 AuthenticatedVoter 来处理它们。

- `IS_AUTHENTICATED_ANONYMOUSLY `表示用户不需要登录就可以访问；通常登录地址需要设置为该值。
- `IS_AUTHENTICATED_REMEMBERED` 表示用户需要是通过 Remember-Me 功能进行自动登录的才能访问；
- `IS_AUTHENTICATED_FULLY `表示用户的认证类型应该是除前两者以外的，也就是用户需要是通过登录入口进行登录认证的才能访问。

```xml
   <security:http>
      <security:form-login login-page="/login.jsp"/>
      <!-- 登录页面可以匿名访问 -->
      <security:intercept-url pattern="/login.jsp*" access="IS_AUTHENTICATED_ANONYMOUSLY"/>
      <security:intercept-url pattern="/**" access="ROLE_USER"/>
   </security:http>
```

> RoleVoter是Spring Security内置的一个AccessDecisionVoter，其会将ConfigAttribute简单的看作是一个角色名称，在投票的时如果拥有该角色即投赞成票。如果ConfigAttribute是以“ROLE_”开头的，则将使用RoleVoter进行投票。当用户拥有的权限中有一个或多个能匹配受保护对象配置的以“ROLE_”开头的ConfigAttribute时其将投赞成票；如果用户拥有的权限中没有一个能匹配受保护对象配置的以“ROLE_”开头的ConfigAttribute，则RoleVoter将投反对票；如果受保护对象配置的ConfigAttribute中没有以“ROLE_”开头的，则RoleVoter将弃权。

## 1.3、指定访问协议

 如果你的应用同时支持Http和Https访问，且要求某些URL只能通过Https访问，这个需求可以通过指定intercept-url的requires-channel属性来指定。requires-channel支持三个值：http、https和any。any表示http和https都可以访问。

```xml
<security:http auto-config="true">
    <security:form-login/>
    <!-- 只能通过https访问 -->
    <security:intercept-url pattern="/admin/**" access="ROLE_ADMIN" requires-channel="https"/>
    <!-- 只能通过http访问 -->
    <security:intercept-url pattern="/**" access="ROLE_USER" requires-channel="http"/>
</security:http>
```

​    需要注意的是当试图使用http请求限制了只能通过https访问的资源时会自动跳转到对应的https通道重新请求。如果所使用的http或者https协议不是监听在标准的端口上（http默认是80，https默认是443），则需要我们通过port-mapping元素定义好它们的对应关系。

```xml
<security:http auto-config="true">
    <security:form-login/>
    <!-- 只能通过https访问 -->
    <security:intercept-url pattern="/admin/**" access="ROLE_ADMIN" requires-channel="https"/>
    <!-- 只能通过http访问 -->
    <security:intercept-url pattern="/**" access="ROLE_USER" requires-channel="http"/>
    <security:port-mappings>
        <security:port-mapping http="8899" https="9988"/>
    </security:port-mappings>
</security:http>
```

## 1.4、指定请求方法

通常我们都会要求某些URL只能通过POST请求，某些URL只能通过GET请求。这些限制Spring Security也已经为我们实现了，通过指定intercept-url的method属性可以限制当前intercept-url适用的请求方式，默认为所有的方式都可以。

```xml
<security:http auto-config="true">
    <security:form-login/>
    <!-- 只能通过POST访问 -->
    <security:intercept-url pattern="/post/**" method="POST"/>
    <!-- 只能通过GET访问 -->
    <security:intercept-url pattern="/**" access="ROLE_USER" method="GET"/>
</security:http>
```

method的可选值有GET、POST、DELETE、PUT、HEAD、OPTIONS和TRACE。