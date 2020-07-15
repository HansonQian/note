# 1、手动实现 IOC 和 AOP

## 1.1、构建一个银行转账界面

![银行转账界面](03-%E6%89%8B%E5%86%99%E5%AE%9E%E7%8E%B0%20IOC%20%E5%92%8C%20AOP/transfer-page.png)

## 1.2、银行转账表结构

```sql
/* 建表 */
CREATE TABLE `t_account` (
 `card_no` varchar(32) DEFAULT NULL COMMENT '卡号',
 `money` int(11) DEFAULT NULL COMMENT '账户余额',
 `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
 PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
/* 初始化数据 */
insert into `t_account` (`card_no`, `money`, `id`) values('1001','300','1');
insert into `t_account` (`card_no`, `money`, `id`) values('1002','100','2');
```

## 1.3、关键代码

### 1.3.1、TransferServlet

```JAVA
@WebServlet(name = "TransferServlet", urlPatterns = {"/transfer"})
public class TransferServlet extends HttpServlet {
    private ITransferService transferService = new TransferService();
    protected void doPost(HttpServletRequest request,
                          HttpServletResponse response) throws IOException {
        // 1、设置请求编码
        request.setCharacterEncoding(StandardCharsets.UTF_8.name());
        String transferFrom = request.getParameter("transferFrom");
        String transferTo = request.getParameter("transferTo");
        String transferMoney = request.getParameter("transferMoney");
        int money = Integer.parseInt(transferMoney);
        DataResult result = new DataResult();
        try {
            // 2、调用service方法
            transferService.transfer(transferFrom, transferTo, money);
            result.setCode(200);
            result.setMessage("SUCCESS");
        } catch (Exception e) {
            e.printStackTrace();
            result.setCode(500);
            result.setMessage(e.toString());
        }
        // 3、响应
        response.setContentType(Constant.APPLICATION_JSON_VALUE);
        response.getWriter().print(GsonUtil.GsonString(result));
    }
    protected void doGet(HttpServletRequest request,
                         HttpServletResponse response) throws IOException {
        doPost(request, response);
    }
}
```

### 1.3.2、TransferService接口及实现类

#### 1.3.2.1、TransferService接口

```java
public interface ITransferService {
    void transfer(String transferFrom, String transferTo, int money) throws Exception;
}
```

#### 1.3.2.2、TransferService接口实现类

```java
public class TransferService implements ITransferService {
    private IAccountDao accountDao = new JdbcAccountDao();
    @Override
    public void transfer(String transferFrom, String transferTo, int money) throws Exception {
        Account form = accountDao.getAccountByCardNo(transferFrom);
        Account to = accountDao.getAccountByCardNo(transferTo);
        form.setMoney(form.getMoney() - money);
        to.setMoney(to.getMoney() + money);
        accountDao.updateAccountByCardNo(form);
        accountDao.updateAccountByCardNo(to);
    }
}
```

### 1.3.3、AccountDao接口及实现类

#### 1.3.3.1、AccountDao接口

```java
public interface IAccountDao {
    Account getAccountByCardNo(String cardNo) throws SQLException;
    int updateAccountByCardNo(Account account) throws SQLException;
}
```

#### 1.3.3.2、AccountDao接口实现类

```java
public class JdbcAccountDao implements IAccountDao {
    @Override
    public Account getAccountByCardNo(String cardNo) throws SQLException {
        Connection conn = JdbcUtil.getConnection();
        String sql = "SELECT * FROM t_account t where t.card_no = ?";
        PreparedStatement preparedStatement = conn.prepareStatement(sql);
        preparedStatement.setString(1, cardNo);
        ResultSet resultSet = preparedStatement.executeQuery();
        Account account = new Account();
        while (resultSet.next()) {
            account.setId(resultSet.getInt("id"));
            account.setMoney(resultSet.getInt("money"));
            account.setCardNo(cardNo);
        }
        preparedStatement.close();
        conn.close();
        return account;
    }
    @Override
    public int updateAccountByCardNo(Account account) throws SQLException {
        Connection conn = JdbcUtil.getConnection();
        String sql = "UPDATE  t_account t SET t.money = ? WHERE t.card_no = ?";
        PreparedStatement preparedStatement = conn.prepareStatement(sql);
        preparedStatement.setInt(1, account.getMoney());
        preparedStatement.setString(2, account.getCardNo());
        int i = preparedStatement.executeUpdate();
        preparedStatement.close();
        conn.close();
        return i;
    }
}
```

## 1.4、源码

参考：[web-day26](git@github.com:HansonQian/java-web-parent.git) 

## 1.5、银行转账程序问题分析

### 1.5.1、问题一：new关键字

`new` 关键字，将业务层的 `TransferService`接口的实现类和持久层的 `AccountDao`接口的实现类耦合在一起，当需要切换持久层接口的实现类，业务层的代码必须要整改，不符合面向接口开发的最优原则。

**解决办法：**

1. 除了 `new` 以外实现实例化对象的方法（反射），将所有的组件配置到xml文件中
2. 使用工厂模式
3. 代码中只需要声明接口，以及提供 `set`方法

### 1.5.2、问题二：没有事务控制

业务层没有事务控制，当程序运行发生了异常，实际情况下后果严重。

**解决办法：**

1. 手动控制JDBC事务，需要注意的是要保证同一个线程使用的数据库连接对象是同一个

## 1.6、案例程序改造

### 1.6.1、实现基于XML的IOC容器

```java
@Slf4j
public class BeanFactory {
    private static final Map<String, Object> beanMap = new HashMap<String, Object>();
    
    public static void init(InputStream inputStream) {
        SAXReader saxReader = new SAXReader();
        try {
            Document document = saxReader.read(inputStream);
            Element rootElement = document.getRootElement();
            List<Element> childElements = rootElement.elements();
            for (Element element : childElements) {
                String idValue = element.attributeValue("id");
                String classValue = element.attributeValue("class");
                Object obj = Class.forName(classValue).newInstance();
                beanMap.put(idValue, obj);
                List<Element> elementList = element.elements();
                for (Element sonEle : elementList) {
                    String nameValue = sonEle.attributeValue("name");
                    String refValue = sonEle.attributeValue("ref");
                    Object refObj = beanMap.get(refValue);
                    String methodName = "set"
                            + nameValue.substring(0, 1).toUpperCase()
                            + nameValue.substring(1);
                    Class<?> objClass = obj.getClass();
                    Class<?> refObjClass = refObj.getClass();
                    Class<?>[] interfaces = refObjClass.getInterfaces();
                    Method method;
                    if (interfaces.length == 0) {
                        method = objClass
                                .getMethod(methodName, refObjClass);
                    }else{
                        method = objClass
                                .getMethod(methodName, interfaces[0]);
                    }
                    method.invoke(obj, refObj);
                }
            }
        } catch (Exception e) {
            log.error("初始化容器失败{}", e.getMessage());
            e.printStackTrace();
        }
    }
    public static Object getBean(String id) {
        return beanMap.get(id);
    }
}
```

### 1.6.2、实现基于JDK动态代理的AOP

```java
public class JDKProxyBeanFactory {
    private TransactionManager transactionManager;
    public void setTransactionManager(TransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }
    public Object getProxy(final Object target) {
        return Proxy.newProxyInstance(
                this.getClass().getClassLoader(), target.getClass().getInterfaces(),
                (Object proxy, Method method, Object[] args) -> {
                    Object result;
                    try {
                        transactionManager.beginTransaction();
                        result = method.invoke(target, args);
                        transactionManager.commitTransaction();
                    } catch (Exception e) {
                        e.printStackTrace();
                        transactionManager.rollbackTransaction();
                        // 向上抛出异常，方便web层捕获
                        throw e.getCause();
                    }
                    return result;
                }
        );
    }
}
```

> 基于CGLIB实现的代理参考 `CGLibProxyBeanFactory`

### 1.6.3、事务管理器

```java
public class TransactionManager {
    private ConnectionUtil connectionUtil;
    public void setConnectionUtil(ConnectionUtil connectionUtil) {
        this.connectionUtil = connectionUtil;
    }
    // 开启事务
    public void beginTransaction() throws SQLException {
        connectionUtil.getCurrentThreadConn().setAutoCommit(false);
    }
    // 提交事务
    public void commitTransaction() throws SQLException {
        connectionUtil.getCurrentThreadConn().commit();
    }
    // 回滚事务
    public void rollbackTransaction() throws SQLException {
        connectionUtil.getCurrentThreadConn().rollback();
    }
}
```

### 1.6.4、数据库链接对象工具

```java
public class ConnectionUtil {
    private ThreadLocal<Connection> threadLocal = new ThreadLocal<>();
    /**
     * 从当前线程获取数据库连接对象
     * @return 连接对象
     */
    public Connection getCurrentThreadConn() {
        // 判断当前线程是否绑定了链接
        Connection connection = threadLocal.get();
        if (null == connection) {
            // 从连接池拿到链接绑定到线程
            connection = DBPoolUtil.getConnection();
            threadLocal.set(connection);
        }
        return connection;
    }
}
```

### 1.6.5、上下文监听器

```java
@Slf4j
@WebListener("ContextLoaderListener")
public class ContextLoaderListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        log.info("Web容器启动...");
        initIoc();
    }
    @Override
    public void contextDestroyed(ServletContextEvent sce) {
        log.info("Web容器销毁...");
    }
    public void initIoc() {
        log.info("上下文初始化");
        InputStream resourceAsStream = Thread.currentThread().getContextClassLoader()
                .getResourceAsStream("beans.xml");
        // 初始化IOC
        BeanFactory.init(resourceAsStream);
        log.info("IOC 容器初始化完成");
    }
}
```

### 1.6.6、配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans>
    <bean id="connectionUtil" class="com.github.transfer.utils.ConnectionUtil"/>

    <bean id="accountDao" class="com.github.transfer.dao.impl.JdbcAccountDao">
        <property name="connectionUtil" ref="connectionUtil"/>
    </bean>

    <bean id="transferService" class="com.github.transfer.service.impl.TransferService">
        <property name="accountDao" ref="accountDao"/>
    </bean>

    <bean id="transactionManager" class="com.github.transfer.core.
                                         transaction.TransactionManager">
        <property name="connectionUtil" ref="connectionUtil"/>
    </bean>

    <bean id="proxyFactory" class="com.github.transfer.core.factory.ProxyFactory">
        <property name="transactionManager" ref="transactionManager"/>
    </bean>
</beans>
```

### 1.6.7、完整版代码

参考：[web-day27](git@github.com:HansonQian/java-web-parent.git)