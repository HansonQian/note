# 1、基于表达式的权限控制

Spring Security允许我们在定义URL访问或方法访问所应有的权限时使用Spring EL表达式，在定义所需的访问权限时如果对应的表达式返回结果为true则表示拥有对应的权限，反之则无。Spring Security可用表达式对象的基类是SecurityExpressionRoot，其为我们提供了如下在使用Spring EL表达式对URL或方法进行权限控制时通用的内置表达式。

| 表达式                         | 描述                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| hasRole([role])                | 当前用户是否拥有指定角色                                     |
| hasAnyRole([role1,role2])      | 多个角色是一个以逗号进行分隔的字符串。如果当前用户拥有指定角色中的任意一个则返回true |
| hasAuthority([auth])           | 等同于hasRole                                                |
| hasAnyAuthority([auth1,auth2]) | 等同于hasAnyRole                                             |
| Principle                      | 代表当前用户的principle对象                                  |
| authentication                 | 直接从SecurityContext获取的当前Authentication对象            |
| permitAll                      | 总是返回true，表示允许所有的                                 |
| denyAll                        | 总是返回false，表示拒绝所有的                                |
| isAnonymous()                  | 当前用户是否是一个匿名用户                                   |
| isRememberMe()                 | 表示当前用户是否是通过Remember-Me自动登录的                  |
| isAuthenticated()              | 表示当前用户是否已经登录认证成功了                           |
| isFullyAuthenticated()         | 如果当前用户既不是一个匿名用户，同时又不是通过Remember-Me自动登录的，则返回true |

## 1.1、通过表达式控制URL权限

URL的访问权限是通过http元素下的intercept-url元素进行定义的，其access属性用来定义访问配置属性。默认情况下该属性值只能是以字符串进行分隔的字符串列表，且每一个元素都对应着一个角色，因为默认使用的是RoleVoter。通过设置http元素的use-expressions=”true”可以启用intercept-url元素的access属性对Spring EL表达式的支持，use-expressions的值默认为false。启用access属性对Spring EL表达式的支持后每个access属性值都应该是一个返回结果为boolean类型的表达式，当表达式返回结果为true时表示当前用户拥有访问权限。此外WebExpressionVoter将加入AccessDecisionManager的AccessDecisionVoter列表，所以如果不使用NameSpace时应当手动添加WebExpressionVoter到AccessDecisionVoter。

```xml
<security:http use-expressions="true">
    <security:form-login/>
    <security:logout/>
    <security:intercept-url pattern="/**" access="hasRole('ROLE_USER')" />
</security:http>
```

在上述配置中我们定义了只有拥有ROLE_USER角色的用户才能访问系统。

使用表达式控制URL权限使用的表达式对象类是继承自SecurityExpressionRoot的WebSecurityExpressionRoot类。其相比基类而言新增了一个表达式hasIpAddress。hasIpAddress可用来限制只有指定IP或指定范围内的IP才可以访问。

```xml
<security:http use-expressions="true">
    <security:form-login/>
    <security:logout/>
    <security:intercept-url pattern="/**" access="hasRole('ROLE_USER') and hasIpAddress('10.10.10.3')" />
</security:http>
```

在上面的配置中我们限制了只有IP为`”10.10.10.3”`，且拥有ROLE_USER角色的用户才能访问。hasIpAddress是通过Ip地址或子网掩码来进行匹配的。如果要设置10.10.10下所有的子网都可以使用，那么我们对应的hasIpAddress的参数应为`“10.10.10.n/24”`，其中n可以是合法IP内的任意值。具体规则可以参照hasIpAddress()表达式用于比较的IpAddressMatcher的matches方法源码。以下是IpAddressMatcher的源码：

```java
public final class IpAddressMatcher implements RequestMatcher {
    private final int nMaskBits;
    private final InetAddress requiredAddress;

    public IpAddressMatcher(String ipAddress) {
        if (ipAddress.indexOf('/') > 0) {
            String[] addressAndMask = StringUtils.split(ipAddress, "/");
            ipAddress = addressAndMask[0];
            nMaskBits = Integer.parseInt(addressAndMask[1]);
        } else {
            nMaskBits = -1;
        }
        requiredAddress = parseAddress(ipAddress);
    }

    public boolean matches(HttpServletRequest request) {
        return matches(request.getRemoteAddr());
    }

    public boolean matches(String address) {
        InetAddress remoteAddress = parseAddress(address);

        if (!requiredAddress.getClass().equals(remoteAddress.getClass())) {
            return false;
        }

        if (nMaskBits < 0) {
            return remoteAddress.equals(requiredAddress);
        }

        byte[] remAddr = remoteAddress.getAddress();
        byte[] reqAddr = requiredAddress.getAddress();

        int oddBits = nMaskBits % 8;
        int nMaskBytes = nMaskBits/8 + (oddBits == 0 ? 0 : 1);
        byte[] mask = new byte[nMaskBytes];

        Arrays.fill(mask, 0, oddBits == 0 ? mask.length : mask.length - 1, (byte)0xFF);

        if (oddBits != 0) {
            int finalByte = (1 << oddBits) - 1;
            finalByte <<= 8-oddBits;
            mask[mask.length - 1] = (byte) finalByte;
        }
        for (int i=0; i < mask.length; i++) {
            if ((remAddr[i] & mask[i]) != (reqAddr[i] & mask[i])) {
                return false;
            }
        }

        return true;
    }

    private InetAddress parseAddress(String address) {
        try {
            return InetAddress.getByName(address);
        } catch (UnknownHostException e) {
            throw new IllegalArgumentException("Failed to parse address" + address, e);
        }
    }
}
```



## 1.2、通过表达式控制方法权限

Spring Security中定义了四个支持使用表达式的注解，分别是`@PreAuthorize`、`@PostAuthorize`、`@PreFilter`和`@PostFilter`。其中前两者可以用来在方法调用前或者调用后进行权限检查，后两者可以用来对集合类型的参数或者返回值进行过滤。要使它们的定义能够对我们的方法的调用产生影响我们需要设置global-method-security元素的`pre-post-annotations=”enabled”`，默认为`disabled`。

```xm
<security:global-method-security pre-post-annotations="disabled"/>
```

### 1.2.1、使用@PreAuthorize和@PostAuthorize进行访问控制

 @PreAuthorize可以用来控制一个方法是否能够被调用。

```java
@Service
public class UserService implements IUserService {
   @PreAuthorize("hasRole('ROLE_ADMIN')")
   public void addUser(User user) {
      System.out.println("addUser................" + user);
   }
   @PreAuthorize("hasRole('ROLE_USER') or hasRole('ROLE_ADMIN')")
   public User find(int id) {
      System.out.println("find user by id............." + id);
      return null;
   }
}
```

在上面的代码中我们定义了只有拥有角色ROLE_ADMIN的用户才能访问adduser()方法，而访问find()方法需要有ROLE_USER角色或ROLE_ADMIN角色。使用表达式时我们还可以在表达式中使用方法参数：

```java
public class UserService implements IUserService {
   /**
    * 限制只能查询Id小于10的用户
    */
   @PreAuthorize("#id<10")
   public User find(int id) {
      System.out.println("find user by id........." + id);
      return null;
   }
   /**
    * 限制只能查询自己的信息
    */
   @PreAuthorize("principal.username.equals(#username)")
   public User find(String username) {
      System.out.println("find user by username......" + username);
      return null;
   }
   /**
    * 限制只能新增用户名称为abc的用户
    */
   @PreAuthorize("#user.name.equals('abc')")
   public void add(User user) {
      System.out.println("addUser............" + user);
   }
}
```

在上面代码中我们定义了调用find(int id)方法时，只允许参数id小于10的调用；调用find(String username)时只允许username为当前用户的用户名；定义了调用add()方法时只有当参数user的name为abc时才可以调用。

有时候可能你会想在方法调用完之后进行权限检查，这种情况比较少，但是如果你有的话，Spring Security也为我们提供了支持，通过@PostAuthorize可以达到这一效果。使用@PostAuthorize时我们可以使用内置的表达式returnObject表示方法的返回值。我们来看下面这一段示例代码：

```java
@PostAuthorize("returnObject.id%2==0")
public User find(int id) {
    User user = new User();
    user.setId(id);
    return user;
}
```

上面这一段代码表示将在方法find()调用完成后进行权限检查，如果返回值的id是偶数则表示校验通过，否则表示校验失败，将抛出AccessDeniedException。    需要注意的是@PostAuthorize是在方法调用完成后进行权限检查，它不能控制方法是否能被调用，只能在方法调用完成后检查权限决定是否要抛出AccessDeniedException。

### 1.2.2、使用@PreFilter和@PostFilter进行过滤

使用@PreFilter和@PostFilter可以对集合类型的参数或返回值进行过滤。使用@PreFilter和@PostFilter时，Spring Security将移除使对应表达式的结果为false的元素。

```java
@PostFilter("filterObject.id%2==0")
public List<User> findAll() {
    List<User> userList = new ArrayList<User>();
    User user;
    for (int i=0; i<10; i++) {
        user = new User();
        user.setId(i);
        userList.add(user);
    }
    return userList;
}
```

上述代码表示将对返回结果中id不为偶数的user进行移除。filterObject是使用@PreFilter和@PostFilter时的一个内置表达式，表示集合中的当前对象。当@PreFilter标注的方法拥有多个集合类型的参数时，需要通过@PreFilter的filterTarget属性指定当前@PreFilter是针对哪个参数进行过滤的。如下面代码就通过filterTarget指定了当前@PreFilter是用来过滤参数ids的。

```java
@PreFilter(filterTarget="ids", value="filterObject%2==0")
public void delete(List<Integer> ids, List<String> usernames) {
	//...
}
```

## 1.3、使用hasPermission表达式

Spring Security为我们定义了hasPermission的两种使用方式，它们分别对应着PermissionEvaluator的两个不同的hasPermission()方法。Spring Security默认处理Web、方法的表达式处理器分别为DefaultWebSecurityExpressionHandler和DefaultMethodSecurityExpressionHandler，它们都继承自AbstractSecurityExpressionHandler，其所持有的PermissionEvaluator是DenyAllPermissionEvaluator，其对于所有的hasPermission表达式都将返回false。所以当我们要使用表达式hasPermission时，我们需要自已手动定义SecurityExpressionHandler对应的bean定义，然后指定其PermissionEvaluator为我们自己实现的PermissionEvaluator，然后通过global-method-security元素或http元素下的expression-handler元素指定使用的SecurityExpressionHandler为我们自己手动定义的那个bean。

接下来看一个自己实现PermissionEvaluator使用hasPermission()表达式的简单示例。

首先实现自己的PermissionEvaluator，如下所示：

```java
public class CustomizePermissionEvaluator implements PermissionEvaluator {
   public boolean hasPermission(Authentication authentication,
         Object targetDomainObject, Object permission) {
      if ("user".equals(targetDomainObject)) {
         return this.hasPermission(authentication, permission);
      }
      return false;
   }
   /**
    * 总是认为有权限
    */
   public boolean hasPermission(Authentication authentication,
         Serializable targetId, String targetType, Object permission) {
      return true;
   }
   /**
    * 简单的字符串比较，相同则认为有权限
    */
   private boolean hasPermission(Authentication authentication, Object permission) {
      Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
      for (GrantedAuthority authority : authorities) {
         if (authority.getAuthority().equals(permission)) {
            return true;
         }
      }
      return false;
   }
}
```

接下来在ApplicationContext中显示的配置一个将使用PermissionEvaluator的SecurityExpressionHandler实现类，然后指定其所使用的PermissionEvaluator为我们自己实现的那个。这里我们选择配置一个针对于方法调用使用的表达式处理器，DefaultMethodSecurityExpressionHandler，具体如下所示：

```xml
<bean id="expressionHandler" class="org.springframework.security.access.expression.method.
       DefaultMethodSecurityExpressionHandler">
      <property name="permissionEvaluator" ref="myPermissionEvaluator" />
</bean>
  <!-- 自定义的PermissionEvaluator实现 -->
<bean id="customizePermissionEvaluator" class="com.xxx.CustomizePermissionEvaluator"/>
```

有了SecurityExpressionHandler之后，我们还要告诉Spring Security，在使用SecurityExpressionHandler时应该使用我们显示配置的那个，这样我们自定义的PermissionEvaluator才能起作用。因为我们上面定义的是针对于方法的SecurityExpressionHandler，所以我们要指定在进行方法权限控制时应该使用它来进行处理，同时注意设置`pre-post-annotations=”true”`以启用对支持使用表达式的@PreAuthorize等注解的支持。

```xml
<security:global-method-security pre-post-annotations="enabled">
    <security:expression-handler ref="expressionHandler" />
</security:global-method-security>
```

之后我们就可以在需要进行权限控制的方法上使用@PreAuthorize以及hasPermission()表达式进行权限控制了。

```java
@Service
public class UserService implements IUserService {
   /**
    * 将使用方法hasPermission(Authentication authentication,
         Object targetDomainObject, Object permission)进行验证。
    */
   @PreAuthorize("hasPermission('user', 'ROLE_USER')")
   public User find(int id) {
      return null;
   }
   /**
    * 将使用PermissionEvaluator的第二个方法，即hasPermission(Authentication authentication,
         Serializable targetId, String targetType, Object permission)进行验证。
    */
   @PreAuthorize("hasPermission('targetId','targetType','permission')")
   public User find(String username) {
      return null;
   }
   @PreAuthorize("hasPermission('user', 'ROLE_ADMIN')")
   public void add(User user) {
   }
}
```

 在上面的配置中，find(int id)和add()方法将使用PermissionEvaluator中接收三个参数的hasPermission()方法进行验证，而find(String username)方法将使用四个参数的hasPermission()方法进行验证。因为hasPermission()表达式与PermissionEvaluator中hasPermission()方法的对应关系就是在hasPermission()表达式使用的参数基础上加上当前Authentication对象调用对应的hasPermission()方法进行验证。

