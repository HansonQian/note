# 1、Spring Security 关于登录

## 1.1、form-login标签介绍

`security:http`标签下的`security:form-login`子标签是用来定义表单登录信息的。当什么属性都不指定的时候Spring Security会生成一个默认的登录页面。如果不想使用默认的登录页面，可以指定自定义的登录界面。

### 1.1.1、使用自定义登录界面

自定义登录界面是通过``security:form-login``标签的`login-page`属性来指定的，此外还有其他几个属性

- `username-parameter`：表示登录时用户名使用的是哪个参数，默认是 `j_username`
- `password-paramete`：表示登录时密码使用的是哪个参数，默认是`j_password`
- `login-processing-url`：表示登录时提交的地址，默认是`/j-spring-security-check`这个只是 Spring Security 用来标记登录页面使用的提交地址，真正关于登录这个请求是不需要用户自己处理的

所以，我们可以通过如下的配置使Spring Security在需要用户登录时跳转到我们自定义的登录页面

```xml
<security:http auto-config="true">
    <security:intercept-url pattern="/login.jsp" access="IS_AUTHENTICATED_ANONYMOUSLY"/>
    <security:intercept-url pattern="/**" access="ROLE_USER"/>
    <security:form-login login-page="/login.jsp"  username-parameter="username"
         password-parameter="password"  default-target-url="/login.do"/>
</security:http>
```

需要注意的是，之前配置的是所有的请求都需要`ROLE_USER`权限，这意味着自定义的`/login.jsp`也需要该权限，这样就形成了一个死循环了。解决办法是需要给`/login.jsp`放行。通过指定`/login.jsp`的访问权限为`IS_AUTHENTICATED_ANONYMOUSLY`或`ROLE_ANNOYMOUSLY`可以达到这一效果。此外还可以指定`security:http`标签的`security`安全性值为`none`来达到相同的效果。如：

```xml
<security:http security="none" pattern="/login.jsp" />
<security:http auto-config="true">
    <security:form-login login-page="/login.jsp"
                         login-processing-url="/login.do" username-parameter="username"
                         password-parameter="password" />
    <security:intercept-url pattern="/**" access="ROLE_USER" />
</security:http>
```

这两种方式的区别是前者将进入Spring Security定义的一系列用于安全控制的Filter，而后者不会。当指定一个`security:http`标签的`security`属性为`none`时，表示其对应的`pattern`的filter链为空。从Spring 3.1版本开始，Spring Security 允许我们定义多个 `security:http`标签以满足针对不同的 pattern 请求使用不同的 filter 链。当为`security:http`标签指定`pattern`属性时表示对应的`security:http`标签定义将对所有的请求发生作用。

#### 1、自定义登录界面

根据上述的说明，自定义登录界面如下

```html
<form action="login.do" method="post">
    <table>
        <tr>
            <td> 用户名：</td>
            <td><input type="text" name="username"/></td>
        </tr>
        <tr>
            <td> 密码：</td>
            <td><input type="password" name="password"/></td>
        </tr>
        <tr>
            <td colspan="2" align="center">
                <input type="submit" value=" 登录 "/>
                <input type="reset" value=" 重置 "/>
            </td>
        </tr>
    </table>
</form>
```

### 1.1.2、指定登录后界面

#### 1、通过default-target-url指定

默认情况下，我们在登录成功后会返回到原本受限制界面，但如果用户是直接请求登录页面，登录成功后应该跳转到哪里呢？默认情况下它会跳转到当前应用的根路径，即欢迎页面。通过指定`security:form-login`标签的`default-target-url`属性，我们可以让用户在直接登录后跳转到指定的页面。如果想让用户不管是直接请求登录页面，还是通过 Spring Security 引导过来的，登录之后都跳转到指定的页面，我们可以通过指定`security:form-login`标签的`always-use-default-target`属性为 `true`来达到这一效果。

```xml
<security:http auto-config="true">
    <security:intercept-url pattern="/login.jsp" access="IS_AUTHENTICATED_ANONYMOUSLY"/>
    <security:intercept-url pattern="/**" access="ROLE_USER"/>
    <security:form-login login-page="/login.jsp"  username-parameter="username"
         password-parameter="password" always-use-default-target="true"/>
</security:http>
```

#### 2、通过authentication-success-handler-ref指定

`authentication-success-handler-ref` 对应一个 `AuthenticationSuccessHandler`实现类的引用。如果指定了 `authentication-success-handler-ref`，登录认证成功后会调用指定`AuthenticationSuccessHandler`的 `onAuthenticationSuccess` 方法。我们需要在该方法体内对认证成功做一个处理，然后返回对应的认证成功页面。使用了 `authentication-success-handler-ref` 之后认证成功后的处理就由指定的`AuthenticationSuccessHandler`来处理，之前的那些`default-target-url`之类的就都不起作用了。

- AuthenticationSuccessHandler实现类

```java
public class AuthenticationSuccessHandlerImpl implements AuthenticationSuccessHandler {
    @Override
    public void onAuthenticationSuccess(HttpServletRequest request,
                                        HttpServletResponse response,
                                        Authentication authentication) throws IOException, ServletException {
        response.sendRedirect(request.getContextPath());
    }
}
```

- xml配置

```xml
<bean id="authenticationSuccessHandler" class="com.xxx.AuthenticationSuccessHandlerImpl"/>
<security:http auto-config="true">
    <security:intercept-url pattern="/login.jsp" access="IS_AUTHENTICATED_ANONYMOUSLY"/>
    <security:intercept-url pattern="/**" access="ROLE_USER"/>
    <security:form-login login-page="/login.jsp"  username-parameter="username"
                         authentication-success-handler-ref="authenticationSuccessHandler"
                         password-parameter="password"/>
</security:http>
```

### 1.1.3、指定登录失败后的页面

除了可以指定登录认证成功后的页面和对应的 AuthenticationSuccessHandler 之外，form-login 同样允许我们指定认证失败后的页面和对应认证失败后的处理器 AuthenticationFailureHandler。

#### 1、通过 authentication-failure-url 指定

默认情况下登录失败后会返回登录页面，我们也可以通过`security:form-login`标签的`authentication-failure-url`属性来指定登录失败后的页面。需要注意的是登录失败后的页面跟登录页面一样也是需要配置成在未登录的情况下可以访问，否则登录失败后请求失败页面时又会被 Spring Security 重定向到登录页面。

```xml
<security:http auto-config="true">
    <security:intercept-url pattern="/login.jsp" access="IS_AUTHENTICATED_ANONYMOUSLY"/>
    <security:intercept-url pattern="/**" access="ROLE_USER"/>
    <security:form-login login-page="/login.jsp"  username-parameter="username"
         password-parameter="password" default-target-url="/login.do"
                         authentication-failure-url="/login.jsp?error=true"/>
</security:http>
```

#### 2、通过 authentication-failure-handler-ref 指定

类似于`authentication-success-handler-ref`，`authentication-failure-handler-ref `对应一个用于处理认证失败的`AuthenticationFailureHandler` 实现类。指定了该属性，Spring Security 在认证失败后会调用指定 `AuthenticationFailureHandler`的`onAuthenticationFailure`方法对认证失败进行处理，此时`authentication-failure-url`属性将不再发生作用。

- AuthenticationFailureHandler实现类

```java
public class AuthenticationFailureHandlerImpl implements AuthenticationFailureHandler {
    @Override
    public void onAuthenticationFailure(HttpServletRequest request, 
                                        HttpServletResponse response, 
                                        AuthenticationException e) throws IOException, ServletException {
        response.sendRedirect(request.getContextPath());
    }
}
```

- xml配置

```xml
<bean id="authenticationFailureHandler" class="com.xxx.AuthenticationFailureHandlerImpl"/>
<security:http auto-config="true">
    <security:intercept-url pattern="/login.jsp" access="IS_AUTHENTICATED_ANONYMOUSLY"/>
    <security:intercept-url pattern="/**" access="ROLE_USER"/>
    <security:form-login login-page="/login.jsp"  username-parameter="username"
                         authentication-failure-handler-ref="authenticationFailureHandler"
                         password-parameter="password"/>
</security:http>
```

## 1.2、http-basic标签

之前介绍的都是基于 form-login 的表单登录，其实 Spring Security 还支持弹窗进行认证。通过定义 http 元素下的 http-basic 元素可以达到这一效果。

```xml
<security:http auto-config="true">
    <security:http-basic/>
    <security:intercept-url pattern="/**" access="ROLE_USER" />
</security:http>
```

此时，如果我们访问受 Spring Security 保护的资源时，系统将会弹出一个窗口来要求我们进行登录认证。效果如下：

![登录效果](02-Spring%20Security%20%E7%99%BB%E5%BD%95/image-20200608155505367.png)

当然此时我们的表单登录也还是可以使用的，只不过当我们访问受包含资源的时候 Spring Security 不会自动跳转到登录页面。这就需要我们自己去请求登录页面进行登录。

需要注意的是当我们同时定义了 http-basic 和 form-login 元素时，form-login 将具有更高的优先级。即在需要认证的时候 Spring Security 将引导我们到登录页面，而不是弹出一个窗口。