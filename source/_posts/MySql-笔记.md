---
title: MySql 笔记
date: 2020-01-03 13:42:16
tags:
- Mysql
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

<!-- more -->
