# 1、Spring Security 缓存UserDetails

Spring Security提供了一个实现了可以缓存UserDetails的UserDetailsService实现类，CachingUserDetailsService。该类的构造接收一个用于真正加载UserDetails的UserDetailsService实现类。当需要加载UserDetails时，其首先会从缓存中获取，如果缓存中没有对应的UserDetails存在，则使用持有的UserDetailsService实现类进行加载，然后将加载后的结果存放在缓存中。UserDetails与缓存的交互是通过UserCache接口来实现的。CachingUserDetailsService默认拥有UserCache的一个空实现引用，NullUserCache。以下是CachingUserDetailsService的类定义。

```java
public class CachingUserDetailsService implements UserDetailsService {
    private UserCache userCache = new NullUserCache();
    private final UserDetailsService delegate;
	// 使用UserDetailsService进行构造
    CachingUserDetailsService(UserDetailsService delegate) {
        this.delegate = delegate;
    }
    public UserDetails loadUserByUsername(String username) {
        // 先从userCache中加载
        UserDetails user = userCache.getUserFromCache(username);
        if (user == null) {
            // 加载不到，使用UserDetailsService进行加载
            user = delegate.loadUserByUsername(username);
        }
        // 存放缓存
        userCache.putUserInCache(user);
        return user;
    }
    // 省略userCache的getter/setter
}
```

我们可以看到当缓存中不存在对应的UserDetails时将使用引用的UserDetailsService类型的delegate进行加载。加载后再把它存放到Cache中并进行返回。除了NullUserCache之外，Spring Security还为我们提供了一个基于Ehcache的UserCache实现类，EhCacheBasedUserCache，其源码如下所示。

```java
public class EhCacheBasedUserCache implements UserCache, InitializingBean {
    private Ehcache cache;
    public void afterPropertiesSet() throws Exception {
        Assert.notNull(cache, "cache mandatory");
    }
    public UserDetails getUserFromCache(String username) {
        Element element = cache.get(username);
        if (element == null) {
            return null;
        } else {
            return (UserDetails) element.getValue();
        }
    }
    public void putUserInCache(UserDetails user) {
        Element element = new Element(user.getUsername(), user);
        cache.put(element);
    }
    public void removeUserFromCache(UserDetails user) {
        this.removeUserFromCache(user.getUsername());
    }
    public void removeUserFromCache(String username) {
        cache.remove(username);
    }
	// 省略cache的getter/setter
}
```

 从上述源码我们可以看到EhCacheBasedUserCache所引用的Ehcache是空的，所以，当我们需要对UserDetails进行缓存时，我们只需要定义一个Ehcache实例，然后把它注入给EhCacheBasedUserCache就可以了。接下来我们来看一下定义一个支持缓存UserDetails的CachingUserDetailsService的示例。

```xml
<security:authentication-manager alias="authenticationManager">
    <!-- 使用可以缓存 UserDetails 的 CachingUserDetailsService -->
    <security:authentication-provider user-service-ref="cachingUserDetailsService" />
</security:authentication-manager>
<!-- 可以缓存 UserDetails 的 UserDetailsService -->
<bean id="cachingUserDetailsService" class="org.springframework.security.config.authentication.CachingUserDetailsService">
    <!-- 真正加载 UserDetails 的 UserDetailsService -->
    <constructor-arg ref="userDetailsService"/>
    <!-- 缓存 UserDetails 的 UserCache -->
    <property name="userCache">
        <bean class="org.springframework.security.core.userdetails.cache.EhCacheBasedUserCache">
            <!-- 用于真正缓存的 Ehcache 对象 -->
            <property name="cache" ref="ehcache4UserDetails"/>
        </bean>
    </property>
</bean>
<!-- 将使用默认的 CacheManager 创建一个名为 ehcache4UserDetails 的 Ehcache 对象 -->
<bean id="ehcache4UserDetails" class="org.springframework.cache.ehcache.EhCacheFactoryBean"/>
<!-- 从数据库加载 UserDetails 的 UserDetailsService -->
<bean id="userDetailsService"
      class="org.springframework.security.core.userdetails.jdbc.JdbcDaoImpl">
    <property name="dataSource" ref="dataSource" />
</bean>
```

在上面的配置中，我们通过 EhcacheFactoryBean 定义的 Ehcache bean 对象采用的是默认配置，其将使用默认的 CacheManager，即直接通过 CacheManager.getInstance() 获取当前已经存在的 CacheManager 对象，如不存在则使用默认配置自动创建一个，当然这可以通过 cacheManager 属性指定我们需要使用的 CacheManager，CacheManager 可以通过 EhCacheManagerFactoryBean 进行定义。此外，如果没有指定对应缓存的名称，默认将使用 beanName，在上述配置中即为 ehcache4UserDetails，可以通过 cacheName 属性进行指定。此外，缓存的配置信息也都是使用的默认的。