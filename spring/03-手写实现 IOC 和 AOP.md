# 1、手动实现 IOC 和 AOP

## 1.1、构建一个银行转账界面

![银行转账界面](03-%E6%89%8B%E5%86%99%E5%AE%9E%E7%8E%B0%20IOC%20%E5%92%8C%20AOP/transfer-page.png)

## 1.2、银行转账表结构

```sql
CREATE TABLE `t_account` (
  `name` varchar(32) DEFAULT NULL COMMENT '用户名',
  `card_no` int(11) DEFAULT NULL COMMENT '卡号',
  `money` int(11) DEFAULT NULL COMMENT '账户余额',
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

## 1.3、关键代码

### 1.3.1、TransferServlet

