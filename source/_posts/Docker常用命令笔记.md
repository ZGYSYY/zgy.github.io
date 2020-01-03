---
title: Docker常用命令笔记
date: 2020-01-03 11:33:45
tags:
- Docker
categories:
- 后端
- Docker
---

# 使用Dockerfile创建镜像

<!-- more -->

# Docker管理Volume

## 查看所有的volume

```bash
docker volume ls
# 显示所有的volume名称列表
docker volume ls -qf dangling=true
```

## 删除volume

```bash
# 删除指定name的volume
docker volume rm name名称
# 删除所有的volume
docker volume rm $(docker volume ls -qf dangling=true)
```

# 容器与主机之间的数据拷贝

将主机/www/runoob目录拷贝到容器96f7f14e99ab的/www目录下。

```bash
docker cp /www/runoob 96f7f14e99ab:/www/
```

将主机/www/runoob目录拷贝到容器96f7f14e99ab中，目录重命名为www。

```bash
docker cp /www/runoob 96f7f14e99ab:/www
```

将容器96f7f14e99ab的/www目录拷贝到主机的/tmp目录中。

```bash
docker cp  96f7f14e99ab:/www /tmp/
```