---
title: MySql 笔记
date: 2020-01-03 13:42:16
tags:
- MySQL
- 数据库
categories:
- 后端
- 数据库
---

**创建用户**

```sql
CREATE USER 'username'@'localhost' IDENTIFIED BY 'password';
```

**创建数据库**

```sql
CREATE DATABASE 数据库名称; 或者 CREATE SCHEMA 数据库名称;
```

**授权**

```sql
GRANT ALL PRIVILEGES ON 数据库名称.* TO 'username'@'localhost';
```

**删除用户**

```sql
DROP user 'usernaem'@'localhost';
```

**MySql中@ 和 @@的区别**

@x 是 用户自定义的变量 (User variables are written as @var_name)。
@@x 是 global或session变量 (@@global @@session )，默认使用的是@@session.x，session 被省略了。

@@查看全局变量:

```sql
select @@log_error;
select @@FOREIGN_KEY_CKECK;
```

@设置用户自定义变量值:

```sql
SET @t1=0, @t2=0, @t3=0;
SELECT @t1:=(@t2:=1)+@t3:=4,@t1,@t2,@t3;   //5 5 1 4
```

**查看隔离级别**

```sql
select @@tx_isolation;
# MySQL8版本后用以下命令
select @@transaction_isolation;
```

**修改事务隔离级别**

```sql
//设置read uncommitted级别：
set session transaction isolation level read uncommitted;

//设置read committed级别：
set session transaction isolation level read committed;

//设置repeatable read级别：
set session transaction isolation level repeatable read;

//设置serializable级别：
set session transaction isolation level serializable;
```



**查看事务自动提交是否开启**

查看当前会话

```sql
show session VARIABLES LIKE 'autocommit';
```

查看全局

```sql
show global VARIABLES LIKE 'autocommit';
```

**修改事务自动提交**

修改当前会话

```sql
# 0 表示关闭，1 表示开启
set session autocommit=0;
```

修改全局

```sql
# 0 表示关闭，1 表示开启
set global autocommit=0;
```

