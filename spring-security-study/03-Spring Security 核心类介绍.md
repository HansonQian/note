# 1、Spring Security核心类介绍

## 1.1、Authentication

`Authentication`是一个接口，用来表示用户认证信息的，在用户登录认证之前相关信息会封装为一个 Authentication 具体实现类的对象，在登录认证成功之后又会生成一个信息更全面，包含用户权限等信息的 Authentication 对象，然后把它保存在 SecurityContextHolder 所持有的 SecurityContext 中，供后续的程序进行调用，如访问权限的鉴定等。

```java
public interface Authentication extends Principal, Serializable {
	//1.权限集合
	Collection<? extends GrantedAuthority> getAuthorities();
	//2.用户名密码认证时可以理解为密码
	Object getCredentials();
	//3.认证时包含的一些信息。
	Object getDetails();
	//4.用户名密码认证时可理解时用户名
	Object getPrincipal();
	//5.是否被认证，认证为true	
	boolean isAuthenticated();
	//6.设置是否能被认证
	void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException;
}
```

## 1.2、SecurityContextHolder

SecurityContextHolder 是用来保存 SecurityContext 的。SecurityContext 中含有当前正在访问系统的用户的详细信息。默认情况下，SecurityContextHolder 将使用 ThreadLocal 来保存 SecurityContext，这也就意味着在处于同一线程中的方法中我们可以从 ThreadLocal 中获取到当前的 SecurityContext。因为线程池的原因，如果我们每次在请求完成后都将 ThreadLocal 进行清除的话，那么我们把 SecurityContext 存放在 ThreadLocal 中还是比较安全的。这些工作 Spring Security 已经自动为我们做了，即在每一次 request 结束后都将清除当前线程的 ThreadLocal。

SecurityContextHolder 中定义了一系列的静态方法，而这些静态方法内部逻辑基本上都是通过 SecurityContextHolder 持有的 SecurityContextHolderStrategy 来实现的，如 getContext()、setContext()、clearContext()等。而默认使用的 strategy 就是基于 ThreadLocal 的 ThreadLocalSecurityContextHolderStrategy。另外，Spring Security 还提供了两种类型的 strategy 实现，GlobalSecurityContextHolderStrategy 和 InheritableThreadLocalSecurityContextHolderStrategy，前者表示全局使用同一个 SecurityContext，如 C/S 结构的客户端；后者使用 InheritableThreadLocal 来存放 SecurityContext，即子线程可以使用父线程中存放的变量。

一般而言，我们使用默认的 strategy 就可以了，但是如果要改变默认的 strategy，Spring Security 为我们提供了两种方法，这两种方式都是通过改变 strategyName 来实现的。SecurityContextHolder 中为三种不同类型的 strategy 分别命名为 `MODE_THREADLOCAL`、`MODE_INHERITABLETHREADLOCAL` 和 `MODE_GLOBAL`。第一种方式是通过 SecurityContextHolder 的静态方法 setStrategyName() 来指定需要使用的 strategy；第二种方式是通过系统属性进行指定，其中属性名默认为 `spring.security.strategy`，属性值为对应 strategy 的名称。

Spring Security 使用一个 Authentication 对象来描述当前用户的相关信息。SecurityContextHolder 中持有的是当前用户的 SecurityContext，而 SecurityContext 持有的是代表当前用户相关信息的 Authentication 的引用。这个 Authentication 对象不需要我们自己去创建，在与系统交互的过程中，Spring Security 会自动为我们创建相应的 Authentication 对象，然后赋值给当前的 SecurityContext。但是往往我们需要在程序中获取当前用户的相关信息，比如最常见的是获取当前登录用户的用户名。在程序的任何地方，通过如下方式我们可以获取到当前用户的用户名。

```java
public String getCurrentUsername() {
    Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();
    if (principal instanceof UserDetails) {
        return ((UserDetails) principal).getUsername();
    }
    if (principal instanceof Principal) {
        return ((Principal) principal).getName();
    }
    return String.valueOf(principal);
}
```

通过 Authentication.getPrincipal() 可以获取到代表当前用户的信息，这个对象通常是 UserDetails 的实例。获取当前用户的用户名是一种比较常见的需求，关于上述代码其实 Spring Security 在 Authentication 中的实现类中已经为我们做了相关实现，所以获取当前用户的用户名最简单的方式应当如下。

```java
public String getCurrentUsername() {
    return SecurityContextHolder.getContext().getAuthentication().getName();
}
```

此外，调用 SecurityContextHolder.getContext() 获取 SecurityContext 时，如果对应的 SecurityContext 不存在，则 Spring Security 将为我们建立一个空的 SecurityContext 并进行返回。

## 1.3、AuthenticationManager

> 核心验证器

`AuthenticationManager`是一个用来处理认证（Authentication）请求的接口。在其中只定义了一个方法 `authenticate()`，该方法只接收一个代表认证请求的Authentication对象作为参数，如果认证成功，则会返回一个封装了当前用户权限等信息的Authentication对象进行返回。

```java
public interface AuthenticationManager {
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
}
```

在 Spring Security 中，`AuthenticationManager`的默认实现是`ProviderManager`，而且它不直接自己处理认证请求，而是委托给其所配置的`List<AuthenticationProvider>`列表，然后会依次使用每一个AuthenticationProvider进行认证，如果有一个AuthenticationProvider认证后的结果不为`null`，则表示该AuthenticationProvider已经认证成功，之后的 AuthenticationProvider将不再继续认证。然后直接以该AuthenticationProvider的认证结果作为ProviderManager的认证结果。如果所有的AuthenticationProvider的认证结果都为`null`，则表示认证失败，将抛出一个 ProviderNotFoundException。

校验认证请求最常用的方法是根据请求的用户名加载对应的`UserDetails`，然后比对 UserDetails的密码与认证请求的密码是否一致，一致则表示认证通过。Spring Security 内部的`DaoAuthenticationProvider`就是使用的这种方式。其内部使用`UserDetailsService`来负责加载UserDetails。在认证成功以后会使用加载的UserDetails来封装要返回的Authentication对象，加载的UserDetails对象是包含用户权限等信息的。认证成功返回的Authentication对象将会保存在当前的SecurityContext中。

当我们在使用`NameSpace`时，`authentication-manager`元素的使用会使 Spring Security 在内部创建一个 ProviderManager，然后可以通过authentication-provider元素往其中添加AuthenticationProvider。当定义 authentication-provider元素时，如果没有通过`ref`属性指定关联哪个AuthenticationProvider，Spring Security 默认就会使用DaoAuthenticationProvider。使用了NameSpace后我们就不要再声明ProviderManager了。

```xml
<security:authentication-manager alias="authenticationManager">
    <security:authentication-provider
              user-service-ref="userDetailsService"/>
</security:authentication-manager>
```

如果我们没有使用`NameSpace`，那么我们就应该在 ApplicationContext 中声明一个`ProviderManager`，当Spring Security 默认提供的实现类不能满足需求的时候可以扩展`AuthenticationProvider` 覆盖`supports(Class<?> authentication) `方法。

### 1.3.1、认证成功后清除凭证

默认情况下，在认证成功后 ProviderManager 将清除返回的 Authentication 中的凭证信息，如密码。所以如果你在无状态的应用中将返回的 Authentication 信息缓存起来了，那么以后你再利用缓存的信息去认证将会失败，因为它已经不存在密码这样的凭证信息了。所以在使用缓存的时候你应该考虑到这个问题。一种解决办法是设置 ProviderManager 的 eraseCredentialsAfterAuthentication 属性为 false，或者想办法在缓存时将凭证信息一起缓存。

### 1.3.2、ProviderManager 源码

> ProviderManage是 AuthenticationManager 的实现类，提供了基本认证实现逻辑和流程；

```java
public Authentication authenticate(Authentication authentication) throws AuthenticationException {
    	//1.获取当前的Authentication的认证类型
        Class<? extends Authentication> toTest = authentication.getClass();
        AuthenticationException lastException = null;
        Authentication result = null;
        //2.遍历所有的providers使用supports方法判断该provider是否支持当前的认证类型，不支持的话继续遍历
        for (AuthenticationProvider provider : getProviders()) {
            if (!provider.supports(toTest)) {
                continue;
            }
            try {
                //3.支持的话调用provider的authenticat方法认证
                result = provider.authenticate(authentication);
                if (result != null) {
                    //4.认证通过的话重新生成Authentication对应的Token
                    copyDetails(authentication, result);
                    break;
                }
            } catch (AccountStatusException e) {
                prepareException(e, authentication);
                throw e;
            } catch (InternalAuthenticationServiceException e) {
                prepareException(e, authentication);
                throw e;
            } catch (AuthenticationException e) {
                lastException = e;
            }
        }

        if (result == null && parent != null) {
            try {
                //5.如果#1 没有验证通过，则使用父类型AuthenticationManager进行验证
                result = parent.authenticate(authentication);
            } catch (ProviderNotFoundException e) {
            } catch (AuthenticationException e) {
                lastException = e;
            }
        }
		//6. 是否擦出敏感信息
        if (result != null) {
            if (eraseCredentialsAfterAuthentication 
                && (result instanceof CredentialsContainer)) {
                ((CredentialsContainer)result).eraseCredentials();
            }
            eventPublisher.publishAuthenticationSuccess(result);
            return result;
        }
        if (lastException == null) {
            lastException = new ProviderNotFoundException(
                messages.getMessage("ProviderManager.providerNotFound",
                        new Object[] {
                            toTest.getName()}, "No AuthenticationProvider found for {0}"));
        }
        prepareException(lastException, authentication);
        throw lastException;
    }
```

- 遍历所有的 Providers，然后依次执行该 Provider 的验证方法
  - 如果某一个 Provider 验证成功，则跳出循环不再执行后续的验证
  - 如果验证成功，会将返回的 result 既 Authentication 对象进一步封装为 Authentication Token； 比如 UsernamePasswordAuthenticationToken、RememberMeAuthenticationToken 等；这些 Authentication Token 也都继承自 Authentication 对象
- 如果 #1 没有任何一个 Provider 验证成功，则试图使用其 parent Authentication Manager 进行验证
- 是否需要擦除密码等敏感信息

## 1.4、AuthenticationProvider

`ProviderManager` 通过 `AuthenticationProvider` 扩展出更多的验证提供的方式；而 `AuthenticationProvider` 本身也就是一个接口，从类图中我们可以看出它的实现类`AbstractUserDetailsAuthenticationProvider `和`AbstractUserDetailsAuthenticationProvider`的子类`DaoAuthenticationProvider `。`DaoAuthenticationProvider `是`Spring Security`中一个核心的`Provider`，对所有的数据库提供了基本方法和入口。

### 1.4.1、DaoAuthenticationProvider

`DaoAuthenticationProvider`主要做了以下事情：

#### 1.4.1.1、对用户身份信息进行加密

```java
/*
 * 可直接返回BCryptPasswordEncoder，也可以自己实现该接口使用自己的加密算法核心方法
 * String encode(CharSequence rawPassword);和
 * boolean matches(CharSequence rawPassword, String encodedPassword);
 */
private PasswordEncoder passwordEncoder;
```

#### 1.4.1.2、实现 AbstractUserDetailsAuthenticationProvider 两个抽象方法

- 获取用户信息的扩展点

```java
protected final UserDetails retrieveUser(String username,
     UsernamePasswordAuthenticationToken authentication)
     throws AuthenticationException {
 UserDetails loadedUser;

 try {
     loadedUser = this.getUserDetailsService().loadUserByUsername(username);
 }
```

主要是通过注入`UserDetailsService`接口对象，并调用其接口方法 `loadUserByUsername(String username)` 获取得到相关的用户信息。`UserDetailsService`接口非常重要。

- 实现 additionalAuthenticationChecks 的验证方法(主要验证密码)

```java
protected abstract void additionalAuthenticationChecks(UserDetails userDetails,
     UsernamePasswordAuthenticationToken authentication)throws AuthenticationException;
```

### 1.4.2、AbstractUserDetailsAuthenticationProvider 源码

> AbstractUserDetailsAuthenticationProvider 为 DaoAuthenticationProvider 提供了基本的认证方法，`AbstractUserDetailsAuthenticationProvider`主要实现了`AuthenticationProvider`的接口方法` authenticate` 并提供了相关的验证逻辑

```java
public Authentication authenticate(Authentication authentication) 
    throws AuthenticationException {
    String username = (authentication.getPrincipal() == null) ? "NONE_PROVIDED" : authentication.getName();
    boolean cacheWasUsed = true;
    UserDetails user = this.userCache.getUserFromCache(username);
    if (user == null) {
        cacheWasUsed = false;
        try {
            //1.获取用户信息由子类实现即DaoAuthenticationProvider
            user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
        } catch (UsernameNotFoundException notFound) {
            if (hideUserNotFoundExceptions) {
                throw new BadCredentialsException(
                    messages.getMessage(
                    "AbstractUserDetailsAuthenticationProvider.badCredentials",
                        "Bad credentials"));
            } else {
                throw notFound;
            }
        }
    }

    try {
        //2.预检查由DefaultPreAuthenticationChecks类实现（主要判断当前用户是否锁定，过期，冻结User接口）
        preAuthenticationChecks.check(user);
        //3.子类实现
        additionalAuthenticationChecks(user, 
                                       (UsernamePasswordAuthenticationToken) authentication);
    } catch (AuthenticationException exception) {
        if (cacheWasUsed) 
            cacheWasUsed = false;
            user = retrieveUser(username,
                                (UsernamePasswordAuthenticationToken) authentication);
            preAuthenticationChecks.check(user);
            additionalAuthenticationChecks(user, 
                                           (UsernamePasswordAuthenticationToken) authentication);
        } else {
            throw exception;
        }
    }
	//4.后检测用户密码是否过期对应#2 的User接口
    postAuthenticationChecks.check(user);
    if (!cacheWasUsed) {
        this.userCache.putUserInCache(user);
    }
    Object principalToReturn = user;
    if (forcePrincipalAsString) {
        principalToReturn = user.getUsername();
    }
    return createSuccessAuthentication(principalToReturn, authentication, user);
}
```

1. 获取`UserDetails` `AbstractUserDetailsAuthenticationProvider`定义了一个抽象的方法

   ```java
   protected abstract UserDetails retrieveUser(String username,
     UsernamePasswordAuthenticationToken authentication)throws AuthenticationException;
   ```

2. 三步验证工作

   1. preAuthenticationChecks
   2. additionalAuthenticationChecks（抽象方法，子类实现）
   3. postAuthenticationChecks

3. 将已通过验证的用户信息封装成 UsernamePasswordAuthenticationToken 对象并返回；该对象封装了用户的身份信息，以及相应的权限信息，相关源码如下

   ```java
   protected Authentication createSuccessAuthentication(Object principal,
                          Authentication authentication,  UserDetails user) {
       UsernamePasswordAuthenticationToken result =
           new UsernamePasswordAuthenticationToken(principal,
                                 authentication.getCredentials(),                                               authoritiesMapper.mapAuthorities(user.getAuthorities()));
       result.setDetails(authentication.getDetails());
       return result;
   }
   ```

## 1.5、UserDetailsService

通过 Authentication.getPrincipal() 的返回类型是 Object，但很多情况下其返回的其实是一个 UserDetails 的实例。UserDetails 是 Spring Security 中一个核心的接口。其中定义了一些可以获取用户名、密码、权限等与认证相关的信息的方法。Spring Security 内部使用的 UserDetails 实现类大都是内置的 User 类，我们如果要使用 UserDetails 时也可以直接使用该类。在 Spring Security 内部很多地方需要使用用户信息的时候基本上都是使用的 UserDetails，比如在登录认证的时候。登录认证的时候 Spring Security 会通过 `UserDetailsService`的`loadUserByUsername()`方法获取对应的 UserDetails 进行认证，认证通过后会将该 UserDetails 赋给认证通过的 Authentication 的 principal，然后再把该 Authentication 存入到 SecurityContext 中。之后如果需要使用用户信息的时候就是通过 SecurityContextHolder 获取存放在 SecurityContext 中的 Authentication 的 principal。

通常我们需要在应用中获取当前用户的其它信息，如 Email、电话等。此时存放在 Authentication 的 principal 中只包含有认证相关信息的 UserDetails 对象可能就不能满足我们的要求了。但是我们可以实现自己的 UserDetails，在该实现类中我们可以定义一些获取用户其它信息的方法，这样将来我们就可以直接从当前 SecurityContext 的 Authentication 的 principal 中获取这些信息了。上文已经提到了 UserDetails 是通过 UserDetailsService 的 loadUserByUsername() 方法进行加载的。UserDetailsService 也是一个接口，我们也需要实现自己的 UserDetailsService 来加载我们自定义的 UserDetails 信息。然后把它指定给 AuthenticationProvider 即可。如下是一个配置 UserDetailsService 的示例。

```xml
<!-- 用于认证的 AuthenticationManager -->
<security:authentication-manager alias="authenticationManager">
    <security:authentication-provider
              user-service-ref="userDetailsService" />
</security:authentication-manager>

<bean id="userDetailsService"
      class="org.springframework.security.core.userdetails.jdbc.JdbcDaoImpl">
    <property name="dataSource" ref="dataSource" />
</bean>
```

上述代码中我们使用的`JdbcDaoImpl`是 Spring Security 为我们提供的`UserDetailsService`的实现，另外 Spring Security 还为我们提供了 UserDetailsService 另外一个实现，`InMemoryDaoImpl`。其作用是从数据库中加载 UserDetails 信息。其中已经定义好了加载相关信息的默认脚本，这些脚本也可以通过 JdbcDaoImpl 的相关属性进行指定。

### 1.5.1、JdbcDaoImpl

`JdbcDaoImpl`允许我们从数据库来加载 UserDetails，其底层使用的是 Spring 的 JdbcTemplate 进行操作，所以我们需要给其指定一个数据源。此外，我们需要通过 usersByUsernameQuery 属性指定通过 username 查询用户信息的 SQL 语句；通过 authoritiesByUsernameQuery 属性指定通过 username 查询用户所拥有的权限的 SQL 语句；如果我们通过设置 JdbcDaoImpl 的 enableGroups 为 true 启用了用户组权限的支持，则我们还需要通过 groupAuthoritiesByUsernameQuery 属性指定根据 username 查询用户组权限的 SQL 语句。当这些信息都没有指定时，将使用默认的 SQL 语句，默认的 SQL 语句如下所示。

- 根据 username 查询用户信息

```sql
select username, password, enabled from users where username=?
```

- 根据 username 查询用户权限信息

```sql
select username, authority from authorities where username=?
```

- 根据 username 查询用户组权限

```sql
select g.id, g.group_name, ga.authority from groups g, groups_members gm, groups_authorities ga where gm.username=? and g.id=ga.group_id and g.id=gm.group_id
```

使用默认的 SQL 语句进行查询时意味着我们对应的数据库中应该有对应的表和表结构，Spring Security 为我们提供的默认表的创建脚本如下。

- 用户表

```sql
create table users(
      username varchar_ignorecase(50) not null primary key,
      password varchar_ignorecase(50) not null,
      enabled boolean not null);
```

- 权限表

```sql
create table authorities (
      username varchar_ignorecase(50) not null,
      authority varchar_ignorecase(50) not null,
      constraint fk_authorities_users foreign key(username) references users(username));
      create unique index ix_auth_username on authorities (username,authority);
```

- 分组表

```sql
create table groups (
  id bigint generated by default as identity(start with 0) primary key,
  group_name varchar_ignorecase(50) notnull);
```

- 分组权限表

```sql
create table group_authorities (
  group_id bigint notnull,
  authority varchar(50) notnull,
  constraint fk_group_authorities_group foreign key(group_id) references groups(id));
```

- 分组成员表

```sql
create table group_members (
  id bigint generated by default as identity(start with 0) primary key,
  username varchar(50) notnull,
  group_id bigint notnull,
  constraint fk_group_members_group foreign key(group_id) references groups(id));
```

此外，使用 jdbc-user-service 元素时在底层 Spring Security 默认使用的就是 JdbcDaoImpl。

```xml
<security:authentication-manager alias="authenticationManager">
    <security:authentication-provider>
        <!-- 基于 Jdbc 的 UserDetailsService 实现，JdbcDaoImpl -->
        <security:jdbc-user-service data-source-ref="dataSource"/>
    </security:authentication-provider>
</security:authentication-manager>
```

### 1.5.2、InMemoryDaoImpl

InMemoryDaoImpl 主要是测试用的，其只是简单的将用户信息保存在内存中。使用 NameSpace 时，使用 user-service 元素 Spring Security 底层使用的 UserDetailsService 就是 InMemoryDaoImpl。此时，我们可以简单的使用 user 元素来定义一个 UserDetails。

```xml
<security:user-service>
    <security:user name="user" password="user" authorities="ROLE_USER"/>
</security:user-service>
```

如上配置表示我们定义了一个用户 user，其对应的密码为 user，拥有 ROLE_USER 的权限。此外，user-service 还支持通过 properties 文件来指定用户信息，如：

```xml
<security:user-service properties="/WEB-INF/config/users.properties"/>
```

其中属性文件应遵循如下格式：

```properties
username=password,grantedAuthority[,grantedAuthority][,enabled|disabled]
```

所以，对应上面的配置文件，我们的 users.properties 文件的内容应该如下所示：

```properties
#username=password,grantedAuthority[,grantedAuthority][,enabled|disabled]
user=user,ROLE_USER
```

## 1.6、GrantedAuthority

Authentication 的 getAuthorities() 可以返回当前 Authentication 对象拥有的权限，即当前用户拥有的权限。其返回值是一个 GrantedAuthority 类型的数组，每一个 GrantedAuthority 对象代表赋予给当前用户的一种权限。GrantedAuthority 是一个接口，其通常是通过 UserDetailsService 进行加载，然后赋予给 UserDetails 的。

GrantedAuthority 中只定义了一个 getAuthority() 方法，该方法返回一个字符串，表示对应权限的字符串表示，如果对应权限不能用字符串表示，则应当返回 null。

Spring Security 针对 GrantedAuthority 有一个简单实现 SimpleGrantedAuthority。该类只是简单的接收一个表示权限的字符串。Spring Security 内部的所有 AuthenticationProvider 都是使用 SimpleGrantedAuthority 来封装 Authentication 对象。

# 2、时序图

![img](03-Spring%20Security%20%E6%A0%B8%E5%BF%83%E7%B1%BB%E4%BB%8B%E7%BB%8D/core-service-Sequence.png)