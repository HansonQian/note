# 1、Spring Security 退出登录Logout

要实现退出登录的功能我们需要在http元素下定义logout元素，这样Spring Security将自动为我们添加用于处理退出登录的过滤器LogoutFilter到FilterChain。当我们指定了http元素的auto-config属性为true时logout定义是会自动配置的，此时我们默认退出登录的URL为“/j_spring_security_logout”，可以通过logout元素的logout-url属性来改变退出登录的默认地址。

```xml
<security:logout logout-url="/logout.do"/>
```

此外，我们还可以给logout指定如下属性：

| 属性名              | 作用                                                         |
| :------------------ | :----------------------------------------------------------- |
| invalidate-session  | 表示是否要在退出登录后让当前session失效，默认为true。        |
| delete-cookies      | 指定退出登录后需要删除的cookie名称，多个cookie之间以逗号分隔。 |
| logout-success-url  | 指定成功退出登录后要重定向的URL。需要注意的是对应的URL应当是不需要登录就可以访问的。 |
| success-handler-ref | 指定用来处理成功退出登录的LogoutSuccessHandler的引用。       |

