# 1、Spring Security Filter

Spring Security 的底层是通过一系列的 Filter 来管理的，每个 Filter 都有其自身的功能，而且各个 Filter 在功能上还有关联关系，所以它们的顺序也是非常重要的。

## 1.1、Filter顺序

Spring Security已经定义了一些Filter，不管实际应用中你用到了哪些，它们应当保持如下顺序：

1. ChannelProcessingFilter，如果你访问的channel错了，那首先就会在channel之间进行跳转，如http变为https
2. SecurityContextPersistenceFilter，这样的话在一开始进行request的时候就可以在SecurityContextHolder中建立一个SecurityContext，然后在请求结束的时候，任何对SecurityContext的改变都可以被copy到HttpSession
3. ConcurrentSessionFilter，因为它需要使用SecurityContextHolder的功能，而且更新对应session的最后更新时间，以及通过SessionRegistry获取当前的SessionInformation以检查当前的session是否已经过期，过期则会调用LogoutHandler
4. 认证处理机制，如UsernamePasswordAuthenticationFilter，CasAuthenticationFilter，BasicAuthenticationFilter等，以至于SecurityContextHolder可以被更新为包含一个有效的Authentication请求
5. SecurityContextHolderAwareRequestFilter，它将会把HttpServletRequest封装成一个继承自HttpServletRequestWrapper的SecurityContextHolderAwareRequestWrapper，同时使用SecurityContext实现了HttpServletRequest中与安全相关的方法
6. JaasApiIntegrationFilter，如果SecurityContextHolder中拥有的Authentication是一个JaasAuthenticationToken，那么该Filter将使用包含在JaasAuthenticationToken中的Subject继续执行FilterChain
7. RememberMeAuthenticationFilter，如果之前的认证处理机制没有更新SecurityContextHolder，并且用户请求包含了一个Remember-Me对应的cookie，那么一个对应的Authentication将会设给SecurityContextHolder
8. AnonymousAuthenticationFilter，如果之前的认证机制都没有更新SecurityContextHolder拥有的Authentication，那么一个AnonymousAuthenticationToken将会设给SecurityContextHolder
9. ExceptionTransactionFilter，用于处理在FilterChain范围内抛出的AccessDeniedException和AuthenticationException，并把它们转换为对应的Http错误码返回或者对应的页面
10. FilterSecurityInterceptor，保护Web URI，并且在访问被拒绝时抛出异常

## 1.2、添加Filter到FilterChain

 当我们在使用NameSpace时，Spring Security是会自动为我们建立对应的FilterChain以及其中的Filter。但有时我们可能需要添加我们自己的Filter到FilterChain，又或者是因为某些特性需要自己显示的定义Spring Security已经为我们提供好的Filter，然后再把它们添加到FilterChain。使用NameSpace时添加Filter到FilterChain是通过http元素下的`custom-filter`元素来定义的。定义custom-filter时需要我们通过ref属性指定其对应关联的是哪个Filter，此外还需要通过`position`、`before`或者`after`指定该Filter放置的位置。Spring Security对FilterChain中Filter顺序是有严格的规定的。

Spring Security对那些内置的Filter都指定了一个别名，同时指定了它们的位置。我们在定义custom-filter的position、before和after时使用的值就是对应着这些别名所处的位置。如`position=”CAS_FILTER”`就表示将定义的Filter放在CAS_FILTER对应的那个位置，`before=”CAS_FILTER”`就表示将定义的Filter放在CAS_FILTER之前，`after=”CAS_FILTER”`就表示将定义的Filter放在CAS_FILTER之后。此外还有两个特殊的位置可以指定，FIRST和LAST，分别对应第一个和最后一个Filter，如你想把定义好的Filter放在最后，则可以使用`after=”LAST”`。

 接下来我们来看一下Spring Security给我们定义好的FilterChain中Filter对应的位置顺序、它们的别名以及将触发自动添加到FilterChain的元素或属性定义。下面的定义是按顺序的。

| 别名                         | Filter类                                       | 对应元素或属性                              |
| ---------------------------- | ---------------------------------------------- | ------------------------------------------- |
| CHANNEL_FILTER               | ChannelProcessingFilter                        | http/intercept-url@requires-channel         |
| SECURITY_CONTEXT_FILTER      | SecurityContextPersistenceFilter               | http                                        |
| CONCURRENT_SESSION_FILTER    | ConcurrentSessionFilter                        | http/session-management/concurrency-control |
| LOGOUT_FILTER                | LogoutFilter                                   | http/logout                                 |
| X509_FILTER                  | X509AuthenticationFilter                       | http/x509                                   |
| PRE_AUTH_FILTER              | AstractPreAuthenticatedProcessingFilter 的子类 | 无                                          |
| CAS_FILTER                   | CasAuthenticationFilter                        | 无                                          |
| FORM_LOGIN_FILTER            | UsernamePasswordAuthenticationFilter           | http/form-login                             |
| BASIC_AUTH_FILTER            | BasicAuthenticationFilter                      | http/http-basic                             |
| SERVLET_API_SUPPORT_FILTER   | SecurityContextHolderAwareRequestFilter        | http@servlet-api-provision                  |
| JAAS_API_SUPPORT_FILTER      | JaasApiIntegrationFilter                       | http@jaas-api-provision                     |
| REMEMBER_ME_FILTER           | RememberMeAuthenticationFilter                 | http/remember-me                            |
| ANONYMOUS_FILTER             | AnonymousAuthenticationFilter                  | http/anonymous                              |
| SESSION_MANAGEMENT_FILTER    | SessionManagementFilter                        | http/session-management                     |
| EXCEPTION_TRANSLATION_FILTER | ExceptionTranslationFilter                     | http                                        |
| FILTER_SECURITY_INTERCEPTOR  | FilterSecurityInterceptor                      | http                                        |
| SWITCH_USER_FILTER           | SwitchUserFilter                               | 无                                          |

## 1.3、DelegatingFilterProxy

可能你会觉得奇怪，我们在 web 应用中使用 Spring Security 时只在 web.xml 文件中定义了如下这样一个 Filter，为什么你会说是一系列的 Filter 呢？

```xml
<filter>
    <filter-name>springSecurityFilterChain</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
</filter>
<filter-mapping>
    <filter-name>springSecurityFilterChain</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

而且如果你不在 web.xml 文件声明要使用的 Filter，那么 Servlet 容器将不会发现它们，它们又怎么发生作用呢？这就是上述配置中 DelegatingFilterProxy 的作用了。

DelegatingFilterProxy 是 Spring 中定义的一个 Filter 实现类，其作用是代理真正的 Filter 实现类，也就是说在调用 DelegatingFilterProxy 的 doFilter() 方法时实际上调用的是其代理 Filter 的 doFilter() 方法。其代理 Filter 必须是一个 Spring bean 对象，所以使用 DelegatingFilterProxy 的好处就是其代理 Filter 类可以使用 Spring 的依赖注入机制方便自由的使用 ApplicationContext 中的 bean。那么 DelegatingFilterProxy 如何知道其所代理的 Filter 是哪个呢？这是通过其自身的一个叫 `targetBeanName` 的属性来确定的，通过该名称，DelegatingFilterProxy 可以从 WebApplicationContext 中获取指定的 bean 作为代理对象。该属性可以通过在 web.xml 中定义 DelegatingFilterProxy 时通过 init-param 来指定，如果未指定的话将默认取其在 web.xml 中声明时定义的名称。

```xml
<filter>
    <filter-name>springSecurityFilterChain</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
</filter>
<filter-mapping>
    <filter-name>springSecurityFilterChain</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

在上述配置中，DelegatingFilterProxy代理的就是名为SpringSecurityFilterChain的Filter。

需要注意的是被代理的Filter的初始化方法init()和销毁方法destroy()默认是不会被执行的。通过设置DelegatingFilterProxy的targetFilterLifecycle属性为true，可以使被代理Filter与DelegatingFilterProxy具有同样的生命周期

## 1.4、FilterChainProxy

Spring Security底层是通过一系列的Filter来工作的，每个Filter都有其各自的功能，而且各个Filter之间还有关联关系，所以它们的组合顺序也是非常重要的。

使用Spring Security时，DelegatingFilterProxy代理的就是一个FilterChainProxy。一个FilterChainProxy中可以包含有多个FilterChain，但是某个请求只会对应一个FilterChain，而一个FilterChain中又可以包含有多个Filter。当我们使用基于Spring Security的NameSpace进行配置时，系统会自动为我们注册一个名为springSecurityFilterChain类型为FilterChainProxy的bean（这也是为什么我们在使用SpringSecurity时需要在web.xml中声明一个name为springSecurityFilterChain类型为DelegatingFilterProxy的Filter了。），而且每一个http元素的定义都将拥有自己的FilterChain，而FilterChain中所拥有的Filter则会根据定义的服务自动增减。所以我们不需要显示的再定义这些Filter对应的bean了，除非你想实现自己的逻辑，又或者你想定义的某个属性NameSpace没有提供对应支持等。

Spring security允许我们在配置文件中配置多个http元素，以针对不同形式的URL使用不同的安全控制。Spring Security将会为每一个http元素创建对应的FilterChain，同时按照它们的声明顺序加入到FilterChainProxy。所以当我们同时定义多个http元素时要确保将更具有特性的URL配置在前。

```xml
<security:http pattern="/login*.jsp*" security="none"/>
<!-- http 元素的 pattern 属性指定当前的 http 对应的 FilterChain 将匹配哪些 URL，
	如未指定将匹配所有的请求 -->
<security:http pattern="/admin/**">
    <security:intercept-url pattern="/**" access="ROLE_ADMIN"/>
</security:http>
<security:http>
    <security:intercept-url pattern="/**" access="ROLE_USER"/>
</security:http>
```

需要注意的是 http 拥有一个匹配 URL 的 pattern，未指定时表示匹配所有的请求，其下的子元素 intercept-url 也有一个匹配 URL 的 pattern，该 pattern 是在 http 元素对应 pattern 基础上的，也就是说一个请求必须先满足 http 对应的 pattern 才有可能满足其下 intercept-url 对应的 pattern。



## 1.5、Spring Security内置的核心Filter

### 1.5.1、FilterSecurityInterceptor

### 1.5.2、ExceptionTranslationFilter

### 1.5.3、SecurityContextPersistenceFilter

### 1.5.4、UsernamePasswordAuthenticationFilter