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

